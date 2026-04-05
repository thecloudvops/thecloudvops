---
layout: post
title: "DevSecOps en Azure: Cómo integrar seguridad desde el primer código con Terraform"
date: 2026-04-05
categories: [Azure, DevSecOps, Terraform, Seguridad]
tags: [azure, devsecops, terraform, seguridad, github-actions, iac]
image: "/assets/images/devsecops-azure.png"
excerpt: "Descubre cómo aplicar el enfoque Shift-Left integrando herramientas de análisis estático en tus pipelines para bloquear configuraciones inseguras antes de que lleguen a producción."
---

## El Problema: La Seguridad como "Ideas de Último Minuto"

Tradicionalmente, en el desarrollo y despliegue de infraestructura, la seguridad era una validación que ocurría al *final* del ciclo de vida. Desplegábamos los servidores, las redes y las bases de datos en Azure y, semanas después, el equipo de SecOps corría un escaneo que arrojaba cientos de vulnerabilidades.

¿El resultado? Retrasos, fricciones entre equipos y, a veces, incidentes graves en producción (como aquel Storage Account con datos sensibles y acceso público, ¿te suena?).

Aquí es donde nace **DevSecOps** y el concepto de **Shift-Left**: mover las comprobaciones de seguridad lo más a la "izquierda" (al inicio) posible en tu pipeline de Integración y Despliegue Continuos (CI/CD).

## La Solución: Análisis Estático de Código (SAST) para IaC

Cuando utilizamos Infraestructura como Código (IaC) con herramientas como **Terraform** o Bicep, nuestra infraestructura es simplemente texto en un archivo. Esto significa que podemos escanear ese texto en busca de malas prácticas *antes* de siquiera intentar desplegarlo.

En el ecosistema de Terraform, herramientas como **[tfsec](https://github.com/aquasecurity/tfsec)** o **[Checkov](https://checkov.io/)** son los validadores por excelencia.

### Ejemplo de un fallo común en Terraform

Imagina que un desarrollador crea este bloque para levantar una cuenta de almacenamiento en Azure:

```hcl
resource "azurerm_storage_account" "example" {
  name                     = "insecurestorage123"
  resource_group_name      = azurerm_resource_group.example.name
  location                 = azurerm_resource_group.example.location
  account_tier             = "Standard"
  account_replication_type = "LRS"
  
  # Error crítico: Se permite tráfico de red desde cualquier IP
  public_network_access_enabled = true
}
```

Si intentamos hacer un merge de esto o aplicar el `terraform apply`, un atacante podría intentar acceder a los datos de la cuenta desde internet.

## Integrando la validación en GitHub Actions

Con DevSecOps, no confiamos ciegamente en que el desarrollador recordará apagar el acceso público. Lo imponemos sistemáticamente en nuestro pipeline de CI.

A continuación, te muestro cómo configurar un **Workflow de GitHub Actions** que escanea el código con `tfsec` cada vez que alguien hace un *Pull Request* hacia la rama principal (`main`).

```yaml
name: DevSecOps Terraform Pipeline

on:
  pull_request:
    branches:
      - main
    paths:
      - 'terraform/**'

jobs:
  sast-scan:
    name: Terraform Security Scan (tfsec)
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Run tfsec
        uses: aquasecurity/tfsec-pr-commenter-action@v1.2.0
        with:
          github_token: {% raw %}${{ secrets.GITHUB_TOKEN }}{% endraw %}
          working_directory: ./terraform
          # Esto hará que el pipeline falle si tfsec encuentra problemas críticos
          soft_fail: false 
```

### ¿Qué ocurre cuando este pipeline se ejecuta?

Al abrir el Pull Request con el código inseguro de arriba, `tfsec` analizará el archivo y el pipeline **fallará inmediatamente** con un error similar a este:

> ❌ **CRITICAL: azurerm_storage_account.example**
> Cuenta de almacenamiento configurada con acceso público.
> *Sugerencia: Configura `public_network_access_enabled = false` o restringe mediante `network_rules`.*

**La infraestructura insegura nunca llegó a intentarse desplegar.** Hemos capturado el error en la fase de revisión de código (Shift-Left).

## Siguientes Pasos: Reforzando el ecosistema Azure

El análisis de código estático (SAST) es solo una de las capas de defensa en profundidad. Para un modelo de gobernanza DevSecOps sólido y end-to-end en Azure debes complementar el código con:

1. **Azure Policy Enforcement:** Aunque el código pase, Azure en su capa de API debe bloquear despliegues que violen políticas de tu empresa. (De esto hablamos en el [artículo anterior sobre Gobernanza y Landing Zones](/blog/azure-landing-zone-gobernanza)).
2. **Microsoft Defender for Cloud:** Para analizar de forma continua los recursos que *ya están* en ejecución y detectar configuraciones que han derivado (Configuration Drift).
3. **Gestión de Secretos:** Nunca (¡nunca!) guardes conexiones de bases de datos o claves en tu código Terraform. Integra `Azure Key Vault` y usa variables gestionadas en GitHub Secrets u OIDC.

## Conclusión

Delegar la seguridad exclusivamente al final del ciclo de vida o al equipo de SecOps no es escalable en la era Cloud. Introducir DevSecOps educando a tu equipo e integrando en tus pipelines validaciones automáticas, convertirá tus automatizaciones en despliegues no solo rápidos, sino confiables y blindados desde el día cero.
