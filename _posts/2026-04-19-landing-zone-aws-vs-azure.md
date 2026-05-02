---
layout: post
title: "Construyendo una Landing Zone en AWS vs Azure: Similitudes y Diferencias (con Terraform)"
image: "/assets/images/aws-azure-lz.png"
categories:
  - blog
tags:
  - AWS
  - Azure
  - Terraform
  - Cloud Architecture
---

Si has estado diseñando infraestructuras en la nube durante el tiempo suficiente, sabes que una **Landing Zone** es el cimiento de cualquier despliegue escalable y seguro. Tradicionalmente, muchos ingenieros nacen especialistas en una sola nube, pero la realidad Multi-Cloud nos empuja a entender cómo se mapean los conceptos de un proveedor a otro.

Hoy vamos a comparar cómo se construye una Landing Zone en **AWS** vs **Azure** usando Terraform, para que puedas trasladar tus conocimientos sin morir en el intento.

## 1. Organización de Cuentas y Jerarquía

El primer paso de una Landing Zone es decidir cómo organizar los entornos (Desarrollo, Producción, Seguridad, etc.).

*   **En Azure:** Utilizamos los **Management Groups**. Esta estructura de árbol nos permite agrupar múltiples *Subscriptions*. El Management Group raíz suele aplicar las políticas globales de la empresa.
*   **En AWS:** El equivalente directo son las **Organizational Units (OUs)** dentro de AWS Organizations. Cada OU puede contener a su vez múltiples *AWS Accounts* (el equivalente a las Subscriptions de Azure).

Con Terraform, en Azure usas `azurerm_management_group`, mientras que en AWS usas `aws_organizations_organizational_unit`. El concepto es sorprendentemente similar.

## 2. Permisos e Identidad (IAM vs RBAC)

Aquí es donde la curva de aprendizaje se hace más pronunciada:

*   **Azure (RBAC):** Azure Role-Based Access Control está muy centralizado y se suele delegar basándose en grupos de Entra ID (antes Azure AD). Las asignaciones se hacen a nivel de Management Group o Suscripción, y se heredan hacia abajo. Es bastante predictivo.
*   **AWS (IAM):** AWS utiliza un sistema más granular donde casi todo es una política de JSON. Las *IAM Policies* se asocian a Roles, Usuarios o Grupos. Además, AWS tiene un peso enorme en las *Resource-based policies* (como las políticas de un bucket de S3), algo que en Azure existe de forma más dispersa.

## 3. Seguridad y Límites (Policies vs SCPs)

¿Cómo te aseguras de que un desarrollador de tu empresa no despliegue servidores carísimos en una región en China?

*   **Azure Policy:** En Azure aplicamos políticas que auditan o deniegan despliegues. Son increíblemente potentes e incluso pueden inyectar recursos (políticas DeployIfNotExists) o añadir etiquetas (tags) automáticamente.
*   **AWS Service Control Policies (SCPs):** Las SCPs se aplican en las OUs y fijan el "límite máximo de permisos" de las cuentas. Si una SCP prohíbe el servicio EC2, da igual si un usuario dentro de esa cuenta tiene privilegios de Administrador; no podrá instanciar computación.

## 4. Repositorios de Código y Módulos

*   **Azure Verified Modules (AVM):** Recientemente Microsoft ha puesto un gran esfuerzo en los AVMs, consolidando estándares para que puedas instanciar Landing Zones o recursos de forma oficial y con buenas prácticas por defecto.
*   **AWS Landing Zone Accelerator / Control Tower:** AWS confía mucho en AWS Control Tower, que puede desplegar una Landing Zone out-of-the-box. A nivel de Terraform, el ecosistema de *terraform-aws-modules* mantenido por la comunidad (como Anton Babenko) es un tesoro indiscutible.

## Resumen

El diseño subyacente es idéntico: separar por entornos (Cuentas/Suscripciones), delegar redes (Hub & Spoke o Transit Gateway/VPC Peering), centralizar identidad, y limitar el radio de explosión (SCPs o Policies). Aprender las particularidades de Terraform para ambos proveedores te garantizará ser un CloudOps de primer nivel en cualquier ecosistema moderno.
