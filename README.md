# Integración de Observabilidad en Pipelines CI/CD con DevOps

Este repositorio contiene un microservicio backend de gestión de tickets desarrollado en Spring Boot, implementando prácticas de DevOps mediante un pipeline de CI/CD automatizado, análisis de calidad de código y estrategias de observabilidad.

---

## Stack Tecnológico y Dependencias

| Categoría | Tecnología / Herramienta |
| :--- | :--- |
| **Lenguaje** | Java 17 |
| **Framework Base** | Spring Boot 3.3.2 |
| **Módulos Spring** | Spring Boot Actuator, Spring Data JPA |
| **Base de Datos** | PostgreSQL 16 |
| **Contenedorización** | Docker, Docker Compose |
| **CI/CD** | GitHub Actions |
| **Calidad de Código** | SonarCloud, JaCoCo |
| **Observabilidad** | AWS CloudWatch Agent |
| **Infraestructura** | Amazon EC2 t3.micro (Amazon Linux 2023) |

---

## Estrategia de Ramas (GitFlow)

El flujo de trabajo del repositorio se rige bajo el modelo de GitFlow para asegurar entregas estables y revisiones de código estructuradas.

* `main`: Rama de producción. Contiene el código estable e implementado.
* `develop`: Rama de integración para el próximo release.
* `feature/*`: Ramas efímeras para el desarrollo de nuevas características.
* `release/*`: Ramas para preparar y estabilizar el código antes de pasar a `main`.
* `hotfix/*`: Ramas para parches críticos directamente sobre `main`.

**Regla de Protección:** Está estrictamente prohibido hacer push directo a las ramas `main` y `develop`. Todo cambio debe integrarse mediante Pull Requests (PR) aprobados.

---

## Arquitectura del Pipeline CI/CD

El flujo de integración y despliegue continuo está orquestado mediante GitHub Actions y consta de 3 *jobs* encadenados:

1.  **Tests & Análisis:** Ejecuta pruebas unitarias, genera reportes de cobertura con JaCoCo, construye el artefacto JAR y ejecuta el análisis estático en SonarCloud. **Regla estricta:** Si el *Quality Gate* de SonarCloud falla, el pipeline se detiene inmediatamente y no se despliega ningún código.
2.  **Build & Push:** Construye la imagen Docker del microservicio y la publica en GitHub Container Registry (GHCR) etiquetada con el SHA del commit.
3.  **Deploy:** Se conecta al *self-hosted runner* en EC2 y despliega la aplicación utilizando `docker compose up -d` con la nueva imagen generada.

---

## Observabilidad con AWS CloudWatch

El sistema está diseñado para ser monitoreado de manera constante a nivel de aplicación e infraestructura:

* **Spring Boot Actuator:** Expone los endpoints `/actuator/health`, `/actuator/metrics` y `/actuator/info` para consultar el estado interno del microservicio.
* **AWS CloudWatch Agent:** Configurado en la instancia EC2 para recolectar métricas de hardware (CPU, memoria, disco y red) y centralizar la gestión de logs.
* **Gestión de Logs:** El agente captura los logs de Docker desde `/var/lib/docker/containers/*/*.log` y los envía al Log Group `/tickets-app/docker`, aplicando filtros específicos para rastrear eventos de nivel `ERROR` y `WARN`.

---

## Cómo las Métricas Apoyan Decisiones Técnicas

La observabilidad implementada permite tomar decisiones informadas sobre la infraestructura y el código:

| Métrica / Evento | Decisión Técnica / Acción Correctiva |
| :--- | :--- |
| **Uso de CPU alto** | Escalar recursos de la instancia o optimizar procesos del microservicio. |
| **Consumo de Memoria alto** | Ajustar los límites de memoria definidos en el archivo Docker Compose. |
| **Picos de Errores en Logs (`ERROR`/`WARN`)** | Detectar bugs de forma proactiva y priorizar *hotfixes* en el código. |
| **Uso de Disco al 100%** | Ejecutar rutinas de limpieza (`docker system prune`) para eliminar imágenes huérfanas. |
| **Health Check en estado `DOWN`** | Gatillar el reinicio automático del contenedor para recuperar la disponibilidad. |

---

## Dashboard en CloudWatch

Para visualizar el estado del sistema en tiempo real, se cuenta con un Dashboard en AWS CloudWatch que exhibe:

* Consumo de CPU y Memoria de la instancia EC2.
* Tráfico de red (In/Out).
* Gráfico de recuento de errores basado en el filtro de logs de Docker.
* *Nota:* La cobertura de código evaluada por JaCoCo se extrae de los artefactos generados durante la ejecución del pipeline CI/CD en GitHub Actions.

---

## Políticas de Cumplimiento y Seguridad

* **Quality Gate:** Validación estricta con SonarCloud para evitar la inyección de deuda técnica, vulnerabilidades o code smells.
* **Branch Protection:** Bloqueo de commits directos en `main` y `develop`.
* **Análisis de Dependencias:** Ejecución semanal de Dependabot (`.github/dependabot.yml`) para parchar librerías de Maven y GitHub Actions desactualizadas.
* **Mínimos Privilegios en Docker:** El `Dockerfile` está configurado para ejecutar la aplicación con un usuario no-root (`devopsuser`).
* **Gestión de Secretos:** Las credenciales y tokens nunca se exponen en texto plano; son inyectados exclusivamente mediante GitHub Secrets.

---

## Infraestructura de Despliegue

* **Servidor:** Instancia Amazon EC2 `t3.micro`.
* **Sistema Operativo:** Amazon Linux 2023.
* **Orquestación Local:** Docker Compose v5.2.0.
* **Runner CI/CD:** GitHub Self-Hosted Runner ejecutándose como un servicio en segundo plano administrado por `systemd`.
* **Seguridad y Permisos AWS:** La instancia opera bajo el perfil de IAM `LabInstanceProfile`.

---

## Secrets Requeridos

Para que el pipeline de GitHub Actions funcione correctamente, los siguientes secretos deben estar configurados en el repositorio:

* `SONAR_TOKEN`: Token de autenticación de SonarCloud.
* `SONAR_PROJECT_KEY`: Identificador del proyecto en SonarCloud.
* `SONAR_ORGANIZATION`: Nombre de la organización en SonarCloud.
* `DB_PASSWORD`: Contraseña de producción para la base de datos PostgreSQL.

---

## Declaración de Uso de IA

Durante el ciclo de desarrollo de este proyecto, se utilizó el modelo **Gemini** como herramienta de apoyo para la redacción de documentación y asistencia en configuraciones técnicas. No obstante, todas las definiciones arquitectónicas, revisiones de código y decisiones técnicas finales fueron tomadas enteramente por el equipo de desarrollo. 

Criterios de uso alineados con: [https://bibliotecas.duoc.cl/ia](https://bibliotecas.duoc.cl/ia)

---

## Reflexiones Académicas Personales

El desarrollo de este proyecto demuestra que la creación de software es un proceso integral donde la cultura DevOps y la automatización actúan como pilares estratégicos. Desde una perspectiva de análisis y gestión, implementar flujos de integración continua y herramientas de observabilidad trasciende lo puramente tecnológico para convertirse en mecanismos esenciales de administración de riesgos y continuidad operativa. En definitiva, automatizar validaciones y monitorear el sistema de forma proactiva no solo reduce el margen de error, sino que optimiza los recursos, permitiendo que el equipo se enfoque en la mejora continua, la innovación y la generación de valor real para la organización.