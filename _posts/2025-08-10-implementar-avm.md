---
title: "Guía práctica: implementar Terraform Azure Verified Modules para ALZ"
image: "/assets/images/implementar-avm.png"
categories:
  - blog
tags:
  - Terraform
  - Azure
  - Microsoft
  - DevOps
---

## Introducción

¿Nunca has trabajado con la nube? No pasa nada. Imagina que la nube es como una ciudad digital donde puedes construir tus propios edificios (aplicaciones, bases de datos, servidores, etc.). Pero, igual que en una ciudad real, necesitas una base sólida y reglas para que todo funcione bien y sea seguro.

**Azure Landing Zones (ALZ)** son como esos cimientos y normas para construir en Azure, la nube de Microsoft. Te ayudan a empezar bien, sin líos ni riesgos.

**Terraform** es una herramienta que te permite "dibujar" tu ciudad en la nube usando archivos de texto. Así puedes repetir, modificar y compartir tu infraestructura fácilmente.

**Azure Verified Modules** son piezas ya preparadas y revisadas por Microsoft para que puedas construir tu base en Azure sin tener que saberlo todo ni escribirlo desde cero.

En esta guía, te explico paso a paso cómo usar estas piezas para crear tu primer entorno en la nube, aunque nunca lo hayas hecho antes.
## Requisitos previos


Antes de empezar, asegúrate de tener:

1. **Cuenta de Azure** con permisos de *Owner* o *Contributor* en la suscripción.
2. **Terraform instalado** en tu máquina o pipeline (mínimo versión 1.5+).
3. **Azure CLI instalado** y sesión iniciada:
   ```bash
   az login
   ```

## Paso 1: Configurar el proveedor de Azure

Este bloque configura Terraform para que sepa cómo conectarse a Azure. Define la versión de Terraform y del proveedor de Azure que vas a usar, asegurando compatibilidad y estabilidad.

```bash
terraform {
  required_version = ">= 1.5.0"
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.50"
    }
  }
}

provider "azurerm" {
  features {}
}
```

💡 Tip: Siempre fija una versión de proveedor para evitar cambios inesperados en el futuro.

## Paso 2: Usar el módulo verificado de Terraform para ALZ

Aquí añades el módulo oficial de Microsoft para crear una Landing Zone. Este módulo configura automáticamente una estructura segura y organizada en Azure, con redes, políticas y recursos básicos.

```
module "alz" {
  source  = "Azure/landing-zone/azurerm"
  version = "0.2.0" # Cambia por la última versión estable

  # Variables mínimas necesarias
  root_id        = "alz"
  root_name      = "MyALZ"
  location       = "westeurope"
  deploy_connectivity_resources = true
}
```

## Paso 3: Comandos básicos

Estos son los comandos principales de Terraform que necesitas ejecutar para desplegar tu infraestructura. Primero inicializas, luego planificas y finalmente aplicas los cambios.

```bash
terraform init    # Descarga proveedores y módulos
terraform plan    # Muestra los cambios que se harán
terraform apply   # Aplica los cambios en Azure
```

⚠️ Advertencia:
Nunca ejecutes terraform apply directamente en producción sin revisar primero el plan.

## Paso 4: Buenas prácticas recomendadas

1. Usar un backend remoto para el estado
Almacena el state file en Azure Blob Storage para compartirlo con tu equipo. Esto evita que el estado se pierda si tu ordenador falla y permite trabajar en equipo sin conflictos.

```
terraform {
  backend "azurerm" {
    resource_group_name  = "tfstate-rg"
    storage_account_name = "tfstateaccount"
    container_name       = "tfstate"
    key                  = "alz.tfstate"
  }
}
```

2. Automatizar con CI/CD
Configura un pipeline en Azure DevOps o GitHub Actions para ejecutar plan y apply automáticamente.

3. Mantener el módulo actualizado
Comprueba periódicamente si hay nuevas versiones y revisa los changelogs antes de actualizar.

4. Usar variables y terraform.tfvars
Evita valores fijos en el código para reutilizar la configuración en distintos entornos.

🎯 Conclusión
Los Azure Verified Modules facilitan enormemente la implementación de Landing Zones, asegurando que desde el primer despliegue cumplas con las buenas prácticas de Microsoft.

Con Terraform, además, ganarás automatización, control de versiones y facilidad de replicar entornos en cuestión de minutos.


💡 ¡Ya tienes tu primera Landing Zone desplegada de forma profesional y segura!