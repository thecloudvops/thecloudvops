---
title: "Platform Engineering: Construyendo el \"Internal Developer Portal\" (IDP) perfecto"
image: "/assets/images/platform-engineering.png"
categories:
  - blog
tags:
  - Platform Engineering
  - DevOps
  - IDP
---

DevOps prometió eliminar el muro entre los desarrolladores (Dev) y los equipos de operaciones (Ops). Sin embargo, en muchas empresas esto resultó en pedir a los desarrolladores que se volvieran expertos en Kubernetes, Terraform, flujos de red y seguridad. ¿El resultado? Sobrecarga cognitiva y frustración.

Aquí es donde entra **Platform Engineering** y su joya de la corona: el Internal Developer Portal (IDP).

## ¿Qué es Platform Engineering?

En lugar de que Operaciones sea un equipo de *tickets* (Jira) que resuelve peticiones, o de que los desarrolladores tengan que construir las tuberías ellos mismos, **Platform Engineering trata la infraestructura y las herramientas internas como un producto.** 

El equipo de plataforma proporciona herramientas de *autoservicio* estandarizadas y seguras, permitiendo a los equipos de desarrollo enfocarse en generar valor de negocio sin preocuparse del "*underlying layer*".

## El Internal Developer Portal (IDP)

Un IDP (como **Spotify Backstage**, Port o Cortex) es la interfaz principal donde interactúan los ingenieros de software de tu empresa.

### ¿Qué debería tener un IDP perfecto?

1. **Golden Paths (Caminos Dorados):** Plantillas aprobadas para empezar un proyecto nuevo. ¿Necesitas un microservicio en Spring Boot con base de datos en Azure SQL y el CI/CD montado? Con un clic en el IDP deberías tener un repositorio generado, los recursos creados vía Terraform en el cloud, y los pipelines listos para el primer *commit*.
2. **Catálogo de Servicios:** Un lugar donde poder ver todos los servicios que existen en la empresa, quién es su propietario ("owner"), repositorios asociados y alertas actuales. Se acabó el preguntar en Slack "*¿De quién es el microservicio de autenticación?*".
3. **Visibilidad de Infraestructura:** Ver de un vistazo cómo se está ejecutando la aplicación, si los Pods de Kubernetes están sanos o si el despliegue falló, sin tener que loguearse a portales de Azure o AWS que muchas veces no entienden o no tienen permisos para usar.
4. **Docs as Code (Documentación):** La consola debería integrar la documentación técnica del servicio renderizando los *markdowns* del código fuente, asegurando que el contexto no languidezca en un sitio web desactualizado.

## El paso final para escalar DevOps

Tratar a tu plataforma interna como un producto implica investigar qué dolores tienen tus desarrolladores (tus verdaderos clientes). 

Si logras implementar un IDP con éxito, no solo acelerarás el Time To Market de cualquier nueva funcionalidad, sino que retendrás asombrosamente el talento técnico de tu compañía al erradicar procesos dolorosos y burocráticos.
