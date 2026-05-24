---
layout: post
title: "Drift Detection en Azure Landing Zones: detecta y corrige desviaciones con Terraform y GitHub Actions"
image: "/assets/images/drift-detection-cover.png"
categories:
  - blog
tags:
  - Azure
  - Terraform
  - Landing Zone
  - GitOps
  - GitHub Actions
  - Gobernanza
  - CAF
---

Tienes tu Landing Zone perfectamente definida en Terraform. Las políticas están como código. El RBAC está controlado. Todo está en Git, revisado y aprobado.

Y entonces alguien entra al portal de Azure, hace un cambio "urgente" directamente, y ya tienes un problema que no sabes que tienes. Tu infraestructura real ya no coincide con tu código. El estado de Terraform ya no refleja la realidad.

Eso es **drift**: la divergencia entre el estado deseado (tu código Terraform) y el estado real (lo que hay en Azure). Y sin un mecanismo activo de detección, puede acumularse durante semanas sin que nadie lo note.

En este artículo vamos a construir un pipeline de GitHub Actions que detecta drift automáticamente, avisa al equipo y genera un informe de qué ha cambiado.

## Por qué el drift es peligroso en una Landing Zone

En una aplicación normal, el drift es un problema de consistencia. En una Landing Zone, es un problema de seguridad y gobernanza:

- Un NSG modificado a mano puede abrir un puerto que debería estar cerrado
- Un role assignment añadido desde el portal rompe el inventario de RBAC
- Una política desactivada temporalmente que nadie recuerda volver a activar

El drift silencioso es exactamente lo que los controles de gobernanza están diseñados para evitar. La ironía es que si no detectas el drift activamente, toda tu inversión en IaC queda comprometida por cambios manuales que se acumulan.

## La arquitectura de detección

El enfoque es sencillo: ejecutar `terraform plan` periódicamente contra el estado real de Azure y analizar si hay diferencias. Si las hay, notificar al equipo con el detalle de qué ha cambiado.

```
GitHub Actions (schedule: diario)
    ↓
terraform init + terraform plan
    ↓
¿Hay cambios?
    ├── NO → Todo OK, log de confirmación
    └── SÍ → Crear GitHub Issue con el diff + notificar a Teams/Slack
```

## Paso 1: Configurar el backend remoto

Para que el pipeline pueda ejecutar `terraform plan`, el estado debe estar en un backend remoto. En Azure, el backend natural es Azure Storage:

```hcl
# providers.tf

terraform {
  backend "azurerm" {
    resource_group_name  = "rg-terraform-state"
    storage_account_name = "stterraformstate"
    container_name       = "tfstate"
    key                  = "landing-zone/terraform.tfstate"
  }

  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0"
    }
  }
}
```

## Paso 2: El workflow de detección de drift

```yaml
# .github/workflows/drift-detection.yml

name: Terraform Drift Detection

on:
  schedule:
    # Ejecutar cada día a las 7:00 UTC (9:00 hora española)
    - cron: '0 7 * * 1-5'
  workflow_dispatch:  # También permitir ejecución manual

permissions:
  contents: read
  issues: write  # Necesario para crear issues automáticamente

env:
  TF_VERSION: '1.7.0'
  WORKING_DIR: './infra'

jobs:
  detect-drift:
    name: Detect Infrastructure Drift
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}
          terraform_wrapper: false

      - name: Azure Login
        uses: azure/login@v1
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}

      - name: Terraform Init
        working-directory: ${{ env.WORKING_DIR }}
        run: terraform init -backend-config="access_key=${{ secrets.TF_STATE_ACCESS_KEY }}"
        env:
          ARM_CLIENT_ID:       ${{ secrets.ARM_CLIENT_ID }}
          ARM_CLIENT_SECRET:   ${{ secrets.ARM_CLIENT_SECRET }}
          ARM_SUBSCRIPTION_ID: ${{ secrets.ARM_SUBSCRIPTION_ID }}
          ARM_TENANT_ID:       ${{ secrets.ARM_TENANT_ID }}

      - name: Terraform Plan (Drift Check)
        id: plan
        working-directory: ${{ env.WORKING_DIR }}
        run: |
          terraform plan \
            -detailed-exitcode \
            -no-color \
            -out=drift.tfplan \
            2>&1 | tee plan_output.txt

          EXIT_CODE=${PIPESTATUS[0]}
          echo "exit_code=$EXIT_CODE" >> $GITHUB_OUTPUT

          # Exit code 0 = no changes, 2 = changes detected, 1 = error
          if [ $EXIT_CODE -eq 1 ]; then
            echo "❌ Terraform plan failed"
            exit 1
          elif [ $EXIT_CODE -eq 2 ]; then
            echo "⚠️ Drift detected!"
            echo "drift_detected=true" >> $GITHUB_OUTPUT
          else
            echo "✅ No drift detected"
            echo "drift_detected=false" >> $GITHUB_OUTPUT
          fi
        env:
          ARM_CLIENT_ID:       ${{ secrets.ARM_CLIENT_ID }}
          ARM_CLIENT_SECRET:   ${{ secrets.ARM_CLIENT_SECRET }}
          ARM_SUBSCRIPTION_ID: ${{ secrets.ARM_SUBSCRIPTION_ID }}
          ARM_TENANT_ID:       ${{ secrets.ARM_TENANT_ID }}

      - name: Upload Plan Output
        if: steps.plan.outputs.drift_detected == 'true'
        uses: actions/upload-artifact@v4
        with:
          name: drift-plan-${{ github.run_id }}
          path: ${{ env.WORKING_DIR }}/plan_output.txt
          retention-days: 30

      - name: Create GitHub Issue on Drift
        if: steps.plan.outputs.drift_detected == 'true'
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const planOutput = fs.readFileSync('${{ env.WORKING_DIR }}/plan_output.txt', 'utf8');

            // Extraer solo las líneas relevantes del plan
            const lines = planOutput.split('\n');
            const summary = lines
              .filter(l => l.match(/^\s*(#|\+|-|~|Plan:)/))
              .slice(0, 50)  // Limitar a 50 líneas para no saturar
              .join('\n');

            const issueBody = `
            ## ⚠️ Infrastructure Drift Detected

            **Date:** ${new Date().toISOString()}
            **Workflow:** [${context.runId}](${context.serverUrl}/${context.repo.owner}/${context.repo.repo}/actions/runs/${context.runId})

            ### Changes detected by Terraform Plan

            \`\`\`hcl
            ${summary}
            \`\`\`

            ### Next steps
            1. Review the full plan output in the [workflow artifacts](${context.serverUrl}/${context.repo.owner}/${context.repo.repo}/actions/runs/${context.runId})
            2. Determine if the change was intentional (→ update Terraform code) or unauthorized (→ revert and investigate)
            3. Apply the Terraform code to restore the desired state if needed

            > This issue was created automatically by the drift detection workflow.
            `;

            // Comprobar si ya existe un issue de drift abierto
            const existingIssues = await github.rest.issues.listForRepo({
              owner: context.repo.owner,
              repo: context.repo.repo,
              state: 'open',
              labels: 'drift-detected'
            });

            if (existingIssues.data.length === 0) {
              await github.rest.issues.create({
                owner: context.repo.owner,
                repo: context.repo.repo,
                title: `🚨 Infrastructure Drift Detected - ${new Date().toLocaleDateString()}`,
                body: issueBody,
                labels: ['drift-detected', 'infrastructure', 'needs-review']
              });
              console.log('Issue created successfully');
            } else {
              // Añadir comentario al issue existente
              await github.rest.issues.createComment({
                owner: context.repo.owner,
                repo: context.repo.repo,
                issue_number: existingIssues.data[0].number,
                body: `### 🔄 Drift still present - ${new Date().toISOString()}\n\n${issueBody}`
              });
              console.log('Comment added to existing drift issue');
            }

      - name: Notify Teams on Drift
        if: steps.plan.outputs.drift_detected == 'true'
        run: |
          curl -s -X POST "${{ secrets.TEAMS_WEBHOOK_URL }}" \
            -H 'Content-Type: application/json' \
            -d '{
              "type": "message",
              "attachments": [{
                "contentType": "application/vnd.microsoft.card.adaptive",
                "content": {
                  "type": "AdaptiveCard",
                  "body": [{
                    "type": "TextBlock",
                    "text": "⚠️ Infrastructure Drift Detected in Landing Zone",
                    "weight": "bolder",
                    "size": "medium"
                  }, {
                    "type": "TextBlock",
                    "text": "Terraform detected differences between the desired state (code) and the actual Azure infrastructure. Review the GitHub issue for details.",
                    "wrap": true
                  }],
                  "actions": [{
                    "type": "Action.OpenUrl",
                    "title": "View Workflow",
                    "url": "${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
                  }]
                }
              }]
            }'
```

## Paso 3: Cerrar el issue automáticamente cuando se resuelve el drift

Cuando el equipo aplica los cambios y el drift desaparece, el issue debería cerrarse automáticamente:

```yaml
      - name: Close Drift Issue if Resolved
        if: steps.plan.outputs.drift_detected == 'false'
        uses: actions/github-script@v7
        with:
          script: |
            const existingIssues = await github.rest.issues.listForRepo({
              owner: context.repo.owner,
              repo: context.repo.repo,
              state: 'open',
              labels: 'drift-detected'
            });

            for (const issue of existingIssues.data) {
              await github.rest.issues.createComment({
                owner: context.repo.owner,
                repo: context.repo.repo,
                issue_number: issue.number,
                body: '✅ Drift resolved. No differences detected between Terraform code and Azure infrastructure.'
              });
              await github.rest.issues.update({
                owner: context.repo.owner,
                repo: context.repo.repo,
                issue_number: issue.number,
                state: 'closed'
              });
            }
```

## El proceso operativo

Una vez implantado, el flujo para gestionar el drift es siempre el mismo:

1. **El pipeline detecta drift** → abre un GitHub Issue con el diff y notifica a Teams
2. **El equipo lo revisa** → ¿fue un cambio intencionado o no autorizado?
3. **Si fue intencionado:** actualizar el código Terraform para reflejar el cambio y hacer PR
4. **Si fue no autorizado:** aplicar `terraform apply` para restaurar el estado deseado e investigar quién lo cambió y por qué
5. **El pipeline confirma que no hay drift** → cierra el issue automáticamente

## Conclusión

El drift es el enemigo silencioso de cualquier estrategia de IaC. Puedes tener el código más limpio del mundo, pero si no detectas activamente cuando alguien se salta el proceso, la realidad y el código se van separando poco a poco hasta que dejan de parecerse.

Un pipeline de detección de drift convierte algo reactivo ("descubrimos que esto estaba mal en la auditoría") en algo proactivo ("el sistema nos avisó a las 9:00 que algo había cambiado"). Y esa diferencia, en una Landing Zone de producción, puede significar la diferencia entre un incidente menor y una brecha de seguridad.
