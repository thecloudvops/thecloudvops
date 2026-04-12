---
title: "GitOps puro para Infraestructura: Adiós al \"Terraform Apply\" manual"
image: "/assets/images/gitops-terraform.png"
categories:
  - blog
tags:
  - GitOps
  - Terraform
  - CI/CD
---

Pregunta incómoda del día: ¿Cuántas veces has ejecutado un `terraform apply` este mes directamente desde el terminal de tu ordenador, cruzando los dedos mientras rezas para que tu credencial temporal no caduque a la mitad del despliegue? 

Si la respuesta es superior a cero, tienes una oportunidad de oro para subir de nivel tus procesos implementando **GitOps para Infraestructura**.

## ¿Por qué evitar el *Apply* manual?

Cuando el despliegue se hace desde máquinas locales, aparecen riesgos catastróficos:
- **Estados (State) asíncronos:** No puedes saber seguro si lo que tiene tu compañero en su rama local coincide con el State en la nube.
- **Falta de visibilidad:** Si alguien destruye una base de datos por error, no queda un rastro auditable (Logs de un pipeline o un PR) más allá del audit trail genérico del proveedor Cloud.
- **Dependencia de credenciales sensibles** repartidas por muchos portátiles locales.

## ¿Qué es GitOps aplicado a Terraform?

Git es la fuente única de verdad. Todo cambio propuesto debe pasar por una *Pull Request* (PR). Y solo las acciones automáticas combinadas con esa PR modifican realmente la infraestructura. 

### Herramientas e Implementación

Hoy en día, herramientas como **Atlantis**, integraciones nativas de **HCP Terraform** (antes Terraform Cloud) o workflows mediante **GitHub Actions / GitLab CI** pueden automatizar este flujo a la perfección.

El ciclo de GitOps habitualmente funciona así:
1. **El commit:** Propongo un cambio mutando mis ficheros `.tf` en una rama nueva (ej `feature/add-vm`).
2. **El Plan (Automatizado):** En el momento que abro la Pull Request hacia la rama principal (`main`), mi pipeline se lanza. Éste hace los `init`, baja dependencias, y lanza un `terraform plan`.
3. **El Feedback (Inmediato):** El pipeline coge la salida de ese *plan* y publica un comentario en mi PR de GitHub. Todos en el equipo vemos de un vistazo si se van a borrar recursos o si algo va mal.
4. **La Revisión (Aprobación):** El Tech Lead o mis compañeros validan la PR tras comprobar el output del plan.
5. **El Apply (Automático e Invisible):** Fusionamos la PR ("Merge"). Esto dispara el CI/CD contra la rama *main* e invoca el sagrado `terraform apply -auto-approve` de forma desatendida y segura a través de perfiles "Workload Identity" (sin usar credenciales/secretos). 

## Conclusión

Migrar de un Terraform local hacia un flujo GitOps absoluto reduce la fricción en el control de calidad, te forzará a trabajar sobre módulos más limpios, y —lo más importante— te permitirá dormir muchísimo más tranquilo por las noches.
