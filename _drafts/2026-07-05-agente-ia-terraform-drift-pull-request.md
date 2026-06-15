---
layout: post
title: "Agentes IA en tu pipeline de Terraform: de la detección de drift al Pull Request automático"
image: "/assets/images/ai-pipeline-cover.png"
categories:
  - blog
tags:
  - Azure
  - IA
  - Claude
  - MCP
  - Terraform
  - GitHub Actions
  - DevOps
  - Agentes IA
---

Esta es la tercera y última entrega de la serie sobre IA aplicada a la gestión de Landing Zones en Azure. En el primer artículo montamos el servidor MCP que conecta Claude a nuestra infraestructura. En el segundo, lo usamos para analizar el compliance en lenguaje natural. Hoy cerramos con el caso más ambicioso: **un agente autónomo que detecta drift, entiende el problema y propone el fix**.

No solo detecta. No solo avisa. Propone una solución concreta en código Terraform y abre un Pull Request para que el equipo lo revise.

## El flujo completo

```
GitHub Actions (schedule: diario)
    ↓
Ejecuta terraform plan → detecta cambios
    ↓
Llama a Claude API con el diff + contexto del repositorio
    ↓
Claude analiza el drift y genera el fix en HCL
    ↓
El agente crea una rama, hace commit del fix y abre un PR
    ↓
El equipo revisa y aprueba (o rechaza)
```

Lo importante: **el humano sigue en el bucle**. El agente propone, el equipo decide. No hay `terraform apply` automático. El PR es el mecanismo de control.

## Paso 1: Ampliar el servidor MCP con herramientas de Git

Añadimos al servidor MCP del artículo 1 las herramientas necesarias para interactuar con el repositorio:

```python
# tools/git_tools.py

import subprocess
import os
from mcp.types import Tool, TextContent


def get_git_tools() -> list[Tool]:
    return [
        Tool(
            name="git_create_branch",
            description="Crea una nueva rama en el repositorio de infraestructura",
            inputSchema={
                "type": "object",
                "properties": {
                    "branch_name": {"type": "string"},
                    "repo_path": {"type": "string"}
                },
                "required": ["branch_name", "repo_path"]
            }
        ),
        Tool(
            name="git_commit_and_push",
            description="Hace commit de los cambios en el repositorio y los sube a origin",
            inputSchema={
                "type": "object",
                "properties": {
                    "repo_path": {"type": "string"},
                    "file_path": {"type": "string", "description": "Fichero a añadir al commit"},
                    "commit_message": {"type": "string"}
                },
                "required": ["repo_path", "file_path", "commit_message"]
            }
        ),
        Tool(
            name="read_terraform_file",
            description="Lee el contenido de un fichero Terraform del repositorio",
            inputSchema={
                "type": "object",
                "properties": {
                    "file_path": {"type": "string", "description": "Ruta absoluta al fichero .tf"}
                },
                "required": ["file_path"]
            }
        ),
        Tool(
            name="write_terraform_file",
            description="Escribe contenido en un fichero Terraform (para aplicar un fix propuesto)",
            inputSchema={
                "type": "object",
                "properties": {
                    "file_path": {"type": "string"},
                    "content": {"type": "string", "description": "Nuevo contenido del fichero"}
                },
                "required": ["file_path", "content"]
            }
        )
    ]


async def git_create_branch(branch_name: str, repo_path: str) -> list[TextContent]:
    result = subprocess.run(
        ["git", "checkout", "-b", branch_name],
        capture_output=True, text=True, cwd=repo_path
    )
    if result.returncode != 0:
        return [TextContent(type="text", text=f"Error: {result.stderr}")]
    return [TextContent(type="text", text=f"Branch '{branch_name}' created successfully")]


async def git_commit_and_push(repo_path: str, file_path: str, commit_message: str) -> list[TextContent]:
    subprocess.run(["git", "add", file_path], cwd=repo_path)
    result = subprocess.run(
        ["git", "commit", "-m", commit_message],
        capture_output=True, text=True, cwd=repo_path
    )
    if result.returncode != 0:
        return [TextContent(type="text", text=f"Commit error: {result.stderr}")]

    push = subprocess.run(
        ["git", "push", "origin", "HEAD"],
        capture_output=True, text=True, cwd=repo_path
    )
    return [TextContent(type="text", text=f"Committed and pushed: {commit_message}")]


async def read_terraform_file(file_path: str) -> list[TextContent]:
    try:
        with open(file_path, "r") as f:
            return [TextContent(type="text", text=f.read())]
    except Exception as e:
        return [TextContent(type="text", text=f"Error reading file: {e}")]


async def write_terraform_file(file_path: str, content: str) -> list[TextContent]:
    try:
        with open(file_path, "w") as f:
            f.write(content)
        return [TextContent(type="text", text=f"File written: {file_path}")]
    except Exception as e:
        return [TextContent(type="text", text=f"Error writing file: {e}")]
```

## Paso 2: El workflow de GitHub Actions con el agente

```yaml
# .github/workflows/drift-agent.yml

name: AI Drift Detection Agent

on:
  schedule:
    - cron: '0 7 * * 1-5'
  workflow_dispatch:

permissions:
  contents: write
  pull-requests: write

jobs:
  ai-drift-agent:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          fetch-depth: 0

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: '1.7.0'
          terraform_wrapper: false

      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Install dependencies
        run: pip install anthropic mcp

      - name: Azure Login
        uses: azure/login@v1
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}

      - name: Terraform Init
        run: terraform init
        working-directory: ./infra
        env:
          ARM_CLIENT_ID: ${{ secrets.ARM_CLIENT_ID }}
          ARM_CLIENT_SECRET: ${{ secrets.ARM_CLIENT_SECRET }}
          ARM_SUBSCRIPTION_ID: ${{ secrets.ARM_SUBSCRIPTION_ID }}
          ARM_TENANT_ID: ${{ secrets.ARM_TENANT_ID }}

      - name: Run terraform plan
        id: plan
        run: |
          terraform plan -no-color -detailed-exitcode -out=drift.tfplan 2>&1 | tee plan_output.txt
          EXIT_CODE=${PIPESTATUS[0]}
          echo "exit_code=$EXIT_CODE" >> $GITHUB_OUTPUT
          if [ $EXIT_CODE -eq 2 ]; then
            echo "drift_detected=true" >> $GITHUB_OUTPUT
          else
            echo "drift_detected=false" >> $GITHUB_OUTPUT
          fi
        working-directory: ./infra
        env:
          ARM_CLIENT_ID: ${{ secrets.ARM_CLIENT_ID }}
          ARM_CLIENT_SECRET: ${{ secrets.ARM_CLIENT_SECRET }}
          ARM_SUBSCRIPTION_ID: ${{ secrets.ARM_SUBSCRIPTION_ID }}
          ARM_TENANT_ID: ${{ secrets.ARM_TENANT_ID }}

      - name: Run AI Drift Agent
        if: steps.plan.outputs.drift_detected == 'true'
        run: python scripts/drift_agent.py
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          GITHUB_REPOSITORY: ${{ github.repository }}
          REPO_PATH: ${{ github.workspace }}
          PLAN_OUTPUT_PATH: ${{ github.workspace }}/infra/plan_output.txt
          ARM_CLIENT_ID: ${{ secrets.ARM_CLIENT_ID }}
          ARM_CLIENT_SECRET: ${{ secrets.ARM_CLIENT_SECRET }}
          ARM_SUBSCRIPTION_ID: ${{ secrets.ARM_SUBSCRIPTION_ID }}
          ARM_TENANT_ID: ${{ secrets.ARM_TENANT_ID }}
```

## Paso 3: El agente Python

```python
# scripts/drift_agent.py

import os
import json
import subprocess
from datetime import datetime
import anthropic

client = anthropic.Anthropic(api_key=os.environ["ANTHROPIC_API_KEY"])

REPO_PATH = os.environ["REPO_PATH"]
PLAN_OUTPUT = open(os.environ["PLAN_OUTPUT_PATH"]).read()
GITHUB_TOKEN = os.environ["GITHUB_TOKEN"]
GITHUB_REPO = os.environ["GITHUB_REPOSITORY"]

SYSTEM_PROMPT = """
Eres un experto en infraestructura de Azure y Terraform especializado en Landing Zones.
Tu tarea es analizar un output de terraform plan que muestra drift (diferencias entre el estado
deseado en código y el estado real en Azure), entender qué cambió y por qué, y proponer un fix
en código Terraform que reconcilie el estado.

Reglas importantes:
1. NUNCA sugiereas hacer terraform apply directamente. Siempre propón cambios como código.
2. Si el drift es una adición de recurso no esperada, propón eliminarlo o documentarlo como excepción.
3. Si el drift es una modificación, propón actualizar el código para reflejar el estado deseado.
4. Sé conservador: si no estás seguro del impacto, indica que se necesita revisión humana.
5. Genera el código HCL correcto y completo para el fichero afectado.
"""

def get_affected_files(plan_output: str) -> list[str]:
    """Extrae los ficheros Terraform afectados del output del plan."""
    files = set()
    for line in plan_output.split('\n'):
        if '# ' in line and ('will be' in line or 'must be' in line):
            resource = line.split('# ')[1].split(' ')[0]
            result = subprocess.run(
                ['grep', '-r', resource.split('.')[0], '--include=*.tf', '-l'],
                capture_output=True, text=True, cwd=REPO_PATH
            )
            for f in result.stdout.strip().split('\n'):
                if f:
                    files.add(f)
    return list(files)


def read_terraform_files(file_paths: list[str]) -> str:
    """Lee el contenido de los ficheros Terraform relevantes."""
    content = ""
    for fp in file_paths[:5]:  # Limitar a 5 ficheros para no saturar el contexto
        try:
            with open(os.path.join(REPO_PATH, fp)) as f:
                content += f"\n\n### Fichero: {fp}\n```hcl\n{f.read()}\n```"
        except Exception:
            pass
    return content


def create_branch_and_pr(fix_content: str, affected_file: str, analysis: str):
    """Crea una rama con el fix propuesto y abre un PR."""
    timestamp = datetime.now().strftime("%Y%m%d-%H%M%S")
    branch_name = f"fix/drift-{timestamp}"

    # Crear rama
    subprocess.run(['git', 'config', 'user.email', 'drift-agent@noreply.github.com'], cwd=REPO_PATH)
    subprocess.run(['git', 'config', 'user.name', 'AI Drift Agent'], cwd=REPO_PATH)
    subprocess.run(['git', 'checkout', '-b', branch_name], cwd=REPO_PATH)

    # Escribir el fix
    fix_path = os.path.join(REPO_PATH, affected_file)
    with open(fix_path, 'w') as f:
        f.write(fix_content)

    # Commit y push
    subprocess.run(['git', 'add', fix_path], cwd=REPO_PATH)
    subprocess.run(['git', 'commit', '-m', f'fix(drift): AI-proposed fix for infrastructure drift'], cwd=REPO_PATH)
    subprocess.run(['git', 'push', 'origin', branch_name], cwd=REPO_PATH)

    # Crear PR via GitHub API
    import urllib.request
    pr_body = f"""## 🤖 AI Drift Detection — Proposed Fix

### Analysis
{analysis}

### Changes proposed
- File modified: `{affected_file}`
- Generated by: AI Drift Agent (Claude)
- Triggered by: Scheduled drift detection workflow

### Review checklist
- [ ] The proposed change reflects the desired state (not the drifted state)
- [ ] No unintended side effects
- [ ] Tests pass after applying

> ⚠️ **This PR was generated automatically. Always review before merging.**
"""
    data = json.dumps({
        "title": f"🤖 [Drift Fix] Infrastructure drift detected - {timestamp}",
        "body": pr_body,
        "head": branch_name,
        "base": "master",
        "labels": ["drift-detected", "ai-generated", "needs-review"]
    }).encode()

    req = urllib.request.Request(
        f"https://api.github.com/repos/{GITHUB_REPO}/pulls",
        data=data,
        headers={
            "Authorization": f"Bearer {GITHUB_TOKEN}",
            "Content-Type": "application/json",
            "Accept": "application/vnd.github+json"
        }
    )
    urllib.request.urlopen(req)
    print(f"PR created for branch {branch_name}")


def main():
    print("🤖 AI Drift Agent starting...")

    # Obtener ficheros afectados
    affected_files = get_affected_files(PLAN_OUTPUT)
    terraform_context = read_terraform_files(affected_files)

    # Llamar a Claude para analizar y proponer fix
    messages = [
        {
            "role": "user",
            "content": f"""Analiza este drift de Terraform y propón un fix:

## Terraform Plan Output (drift detected)
```
{PLAN_OUTPUT[:4000]}
```

## Ficheros Terraform actuales relevantes
{terraform_context}

Por favor:
1. Explica qué cambió en Azure respecto al código (2-3 frases)
2. Indica si el drift fue probablemente intencional o accidental
3. Propón el código HCL corregido para el fichero más relevante
4. Indica el nombre exacto del fichero a modificar

Formato de respuesta:
ANÁLISIS: [tu análisis]
FICHERO: [ruta del fichero a modificar]
CÓDIGO:
```hcl
[código completo del fichero]
```
"""
        }
    ]

    response = client.messages.create(
        model="claude-opus-4-5",
        max_tokens=4000,
        system=SYSTEM_PROMPT,
        messages=messages
    )

    reply = response.content[0].text
    print(f"Claude analysis:\n{reply}")

    # Parsear la respuesta
    analysis = ""
    target_file = ""
    fix_code = ""

    if "ANÁLISIS:" in reply:
        analysis = reply.split("ANÁLISIS:")[1].split("FICHERO:")[0].strip()
    if "FICHERO:" in reply:
        target_file = reply.split("FICHERO:")[1].split("CÓDIGO:")[0].strip()
    if "```hcl" in reply:
        fix_code = reply.split("```hcl")[1].split("```")[0].strip()

    if fix_code and target_file:
        create_branch_and_pr(fix_code, target_file, analysis)
        print("✅ PR created successfully")
    else:
        print("⚠️ Could not extract fix from Claude response. Manual review needed.")
        print(f"Analysis: {analysis}")


if __name__ == "__main__":
    main()
```

## El resultado: un PR listo para revisar

Cuando el agente detecta drift, el equipo recibe un Pull Request como este:

```
🤖 [Drift Fix] Infrastructure drift detected - 20260705-070312

ANÁLISIS: Se detectó que el NSG 'nsg-corp-prod-web' fue modificado manualmente
en Azure para añadir una regla de entrada en el puerto 3389 (RDP). Esta modificación
no está reflejada en el código Terraform y probablemente fue accidental o una
acción de emergencia temporal que no se revirtió.

FICHERO: infra/networking/nsgs.tf

[código HCL con la regla eliminada o documentada como excepción]
```

El equipo tiene dos opciones:
1. **Merge del PR** → aplica el fix en el próximo `terraform apply`, eliminando la regla no autorizada
2. **Cerrar el PR** → decidir que el cambio fue intencional y actualizar el código manualmente para documentarlo

En ambos casos, el proceso queda registrado en Git.

## Conclusión: cerrando el ciclo de automatización

Con esta serie hemos recorrido el camino completo:

1. **Gobernanza como código** (políticas, excepciones, RBAC, detección de drift)
2. **Visibilidad centralizada** (compliance dashboard tipo Security Hub)
3. **IA + MCP** como capa de inteligencia que conecta todo

El equipo de plataforma ya no solo define las reglas: tiene un agente que monitoriza que se cumplen, interpreta las anomalías y propone correcciones. El humano sigue siendo el que aprueba, pero el trabajo de detección y propuesta inicial lo hace la IA.

Eso es lo que separa una Landing Zone gestionada de una Landing Zone inteligente.
