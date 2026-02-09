---
title: "Configuración del Entorno para Terraform: Guía Completa"
date: 2025-10-11
categories:
  - blog
tags:
  - Terraform
  - IaC
  - DevOps
  - Automatización
  - Cloud
---

## Introducción

¿Nunca has usado Terraform ni sabes qué es IaC? No te preocupes. **Terraform** es una herramienta que te permite crear y gestionar recursos en la nube (como servidores, redes o bases de datos) usando archivos de texto. Esto se llama "Infraestructura como Código" (IaC), y hace que todo sea más fácil de repetir, modificar y compartir.

En esta guía, te explico desde cero cómo preparar tu ordenador para usar Terraform, aunque nunca hayas trabajado con la nube ni con programación.

## Requisitos Básicos

### 1. Instalar Terraform CLI

Terraform CLI es el programa principal que necesitas para crear y gestionar recursos en la nube desde tu ordenador. Aquí tienes cómo instalarlo en Windows y Linux, y cómo comprobar que la instalación fue exitosa.

**Windows (con Chocolatey):**
```powershell
choco install terraform
```

**Linux (con apt):**
```bash
sudo apt-get update && sudo apt-get install -y terraform
```

**Verificar la instalación:**
```bash
terraform version
```

### 2. Editor de Código

Para escribir y editar tus archivos de Terraform de forma cómoda, se recomienda usar Visual Studio Code junto con la extensión oficial de HashiCorp. Esto te dará ayudas visuales, autocompletado y validación de tu código.

1. Instalar [Visual Studio Code](https://code.visualstudio.com/)
2. Instalar la extensión [HashiCorp Terraform](https://marketplace.visualstudio.com/items?itemName=HashiCorp.terraform)

Beneficios de la extensión:
- Syntax highlighting (colores para el código)
- IntelliSense (sugerencias automáticas)
- Validación de código (te avisa de errores)
- Formato automático
- Referencias y definiciones

### 3. Git

Git te permite llevar un control de los cambios en tu código y colaborar con otras personas. Es fundamental para cualquier proyecto de infraestructura como código.

```powershell
# Windows (Chocolatey)
choco install git

# Linux
sudo apt-get install git
```

### 4. Credenciales del Proveedor Cloud

Para conectarte a Azure o AWS desde Terraform, necesitas instalar las herramientas de línea de comandos y configurar tus credenciales. Esto permite que Terraform se autentique y gestione recursos en la nube.

#### Azure
```bash
# Instalar Azure CLI
# Windows
winget install -e --id Microsoft.AzureCLI

# Linux
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash

# Iniciar sesión
az login
```

#### AWS
```bash
# Instalar AWS CLI
# Windows
msiexec.exe /i https://awscli.amazonaws.com/AWSCLIV2.msi

# Linux
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Configurar credenciales
aws configure
```

## Estructura de Proyecto Recomendada

Organizar tus archivos en carpetas te ayudará a mantener el proyecto limpio y escalable. Aquí tienes un ejemplo de cómo puedes estructurar un proyecto de Terraform, separando los módulos y las variables.

```plaintext
proyecto-terraform/
├── main.tf           # Configuración principal
├── variables.tf      # Definición de variables
├── outputs.tf        # Outputs definidos
├── providers.tf      # Configuración de proveedores
├── backend.tf        # Configuración del backend
└── environments/
    ├── dev/
    │   └── terraform.tfvars
    ├── staging/
    │   └── terraform.tfvars
    └── prod/
        └── terraform.tfvars
```

## Configuración del Backend

El backend es el lugar donde Terraform guarda el estado de tu infraestructura. Usar Azure Storage como backend permite que varios miembros del equipo compartan el mismo estado y evita conflictos.

```hcl
# backend.tf
terraform {
  backend "azurerm" {
    resource_group_name  = "tfstate-rg"
    storage_account_name = "tfstateaccount"
    container_name       = "tfstate"
    key                  = "prod.terraform.tfstate"
  }
}
```

## Extensiones VS Code Recomendadas

1. HashiCorp Terraform
2. Azure Terraform
3. Terraform doc snippets
4. YAML
5. GitLens

## Buenas Prácticas

1. **Control de Versiones**
   - Usar .gitignore para archivos sensibles
   - Commit con mensajes descriptivos
   - Una rama por feature/cambio

2. **Seguridad**
   - No versionar credenciales
   - Usar variables de entorno
   - Implementar least privilege

3. **Organización**
   - Módulos reutilizables
   - Variables bien documentadas
   - Outputs relevantes

4. **Testing**
   - Terraform plan antes de apply
   - Validación de sintaxis
   - Test automatizados

## Configuración de Pre-commit Hooks

Los pre-commit hooks son herramientas que se ejecutan automáticamente antes de cada commit para validar tu código. Esto asegura que tu código Terraform esté bien formateado y sin errores antes de guardarlo en el repositorio.

```bash
# Instalar pre-commit
pip install pre-commit

# Crear .pre-commit-config.yaml
cat << EOF > .pre-commit-config.yaml
repos:
- repo: https://github.com/antonbabenko/pre-commit-terraform
  rev: v1.76.0
  hooks:
    - id: terraform_fmt
    - id: terraform_docs
    - id: terraform_validate
EOF

# Inicializar pre-commit
pre-commit install
```

## Próximos Pasos

1. Crear tu primer proyecto Terraform
2. Configurar un backend remoto
3. Implementar control de versiones
4. Desarrollar módulos reutilizables
5. Integrar con CI/CD

Este entorno te permitirá desarrollar IaC de manera eficiente y siguiendo las mejores prácticas de la industria. En próximos artículos, profundizaremos en cada uno de estos aspectos.
