---
title: "Migrando de Terraform a OpenTofu: Guía Práctica"
categories:
  - blog
tags:
  - Terraform
  - OpenTofu
  - IaC
  - DevOps
  - Cloud
---

## Introducción

Con el reciente cambio en la licencia de Terraform y el nacimiento de OpenTofu como alternativa open source, muchas organizaciones están considerando la migración. Este artículo explora cómo realizar esta transición de manera segura y efectiva.

## ¿Qué es OpenTofu?

OpenTofu es un fork de Terraform que mantiene la compatibilidad con el ecosistema existente mientras garantiza:
- Licencia 100% open source
- Compatibilidad con proveedores actuales
- Desarrollo comunitario activo
- Soporte empresarial

## Proceso de Migración

1. **Preparación del Entorno**:
```bash
# Instalar OpenTofu
curl --proto '=https' --tlsv1.2 -fsSL https://get.opentofu.org/install.sh | bash
```

2. **Actualización del Backend**:
```hcl
terraform {
  backend "s3" {
    bucket = "my-terraform-state"
    key    = "prod/terraform.tfstate"
    region = "us-west-2"
  }
}
```

3. **Verificación de Compatibilidad**:
```bash
# Verificar el estado actual
tofu state list

# Verificar el plan
tofu plan
```

## Cambios en la Sintaxis

La mayoría del código permanece igual:

```hcl
provider "aws" {
  region = "us-east-1"
}

resource "aws_instance" "example" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"

  tags = {
    Name = "OpenTofu Example"
  }
}
```

## Gestión de Estado

```hcl
# Importar recursos existentes
tofu import aws_s3_bucket.example bucket-name

# Mostrar el estado actual
tofu show
```

## Pipeline de CI/CD

```yaml
name: OpenTofu CI/CD

on:
  push:
    branches: [ main ]

jobs:
  terraform:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    
    - name: Setup OpenTofu
      uses: opentofu/setup-opentofu@v1
      with:
        tofu_version: "1.6.0"
    
    - name: OpenTofu Init
      run: tofu init
      
    - name: OpenTofu Plan
      run: tofu plan -out=tfplan
      
    - name: OpenTofu Apply
      run: tofu apply tfplan
```

## Testing y Validación

```hcl
terraform {
  required_version = ">= 1.6.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 4.0"
    }
  }
}
```

## Mejores Prácticas

1. **Versionado de Módulos**:
```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "3.2.0"
  
  name = "my-vpc"
  cidr = "10.0.0.0/16"
}
```

2. **Workspaces**:
```bash
tofu workspace new development
tofu workspace select development
```

## Resolución de Problemas

```bash
# Limpiar estado corrupto
tofu force-unlock LOCK_ID

# Actualizar providers
tofu init -upgrade
```

## Referencias

- [OpenTofu Documentation](https://opentofu.org/docs)
- [Terraform to OpenTofu Migration Guide](https://opentofu.org/docs/migration)
- [OpenTofu GitHub Repository](https://github.com/opentofu/opentofu)
