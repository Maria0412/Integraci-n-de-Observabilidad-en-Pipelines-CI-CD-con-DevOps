
---

## Observabilidad con AWS CloudWatch

El sistema está diseñado para ser monitoreado de manera constante a nivel de aplicación e infraestructura:

* **Spring Boot Actuator:** Expone los endpoints `/actuator/health`, `/actuator/metrics` y `/actuator/info` para consultar el estado interno del microservicio.
* **AWS CloudWatch Agent:** Configurado en la instancia EC2 para recolectar métricas de hardware (CPU, memoria, disco y red) y centralizar la gestión de logs.
* **Gestión de Logs:** El agente captura los logs de Docker desde `/var/lib/docker/containers/*/*.log` y los envía al Log Group `/tickets-app/docker` en CloudWatch Logs.

---

## Cómo las Métricas Apoyan Decisiones Técnicas

La observabilidad implementada permite tomar decisiones informadas sobre la infraestructura y el código:

| Métrica / Evento | Decisión Técnica / Acción Correctiva |
| :--- | :--- |
| **Uso de CPU alto** | Escalar recursos de la instancia o optimizar procesos del microservicio. |
| **Consumo de Memoria alto** | Ajustar los límites de memoria definidos en el archivo Docker Compose. |
| **Errores en Logs** | Detectar bugs de forma proactiva y priorizar *hotfixes* en el código. |
| **Uso de Disco al 100%** | Ejecutar rutinas de limpieza (`docker system prune`) para eliminar imágenes huérfanas. |
| **Health Check en estado `DOWN`** | Gatillar el reinicio automático del contenedor para recuperar la disponibilidad. |

---

## Dashboard en CloudWatch

Se configuró el dashboard `tickets-app-dashboard` en AWS CloudWatch con los siguientes widgets:

| Widget | Métrica | Tipo |
| :--- | :--- | :--- |
| CPU | `cpu_usage_user` + `cpu_usage_system` | Línea |
| Memoria | `mem_used_percent` | Número |
| Disco | `disk_used_percent` | Medidor |
| Red | `net_bytes_recv` + `net_bytes_sent` | Línea |
| Logs de contenedores | Log group `/tickets-app/docker` | Logs |
| Cobertura de Pruebas | Reporte JaCoCo por paquete | Texto |
| Tiempo de Despliegue | Duración de cada job del pipeline | Texto |

---

## Políticas de Cumplimiento y Seguridad

* **Quality Gate:** Validación estricta con SonarCloud para evitar la inyección de deuda técnica, vulnerabilidades o code smells.
* **Branch Protection:** Bloqueo de commits directos en `main`. Todo cambio entra por Pull Request.
* **Análisis de Dependencias:** Ejecución semanal de Dependabot (`.github/dependabot.yml`) para parchear librerías de Maven y GitHub Actions desactualizadas.
* **Mínimos Privilegios en Docker:** El `Dockerfile` está configurado para ejecutar la aplicación con un usuario no-root (`devopsuser`).
* **Gestión de Secretos:** Las credenciales y tokens nunca se exponen en texto plano; son inyectados exclusivamente mediante GitHub Secrets.

---

## Infraestructura de Despliegue

* **Servidor:** Instancia Amazon EC2 `t3.micro`.
* **Sistema Operativo:** Amazon Linux 2023.
* **Orquestación:** Docker Compose v5.2.0.
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

## Evidencias de Funcionamiento

### Evidencia 1: Pipeline CI/CD exitoso
Pipeline ejecutado correctamente con los 3 jobs en verde: Tests & SonarCloud (1m 3s), Build & Push Docker Image (45s) y Deploy with Docker Compose (29s). Duración total: 2m 26s.

<img width="1853" height="666" alt="Pipeline CI/CD exitoso" src="https://github.com/user-attachments/assets/c0a9d8b1-fc20-4197-8fbf-39819d77e7ac" />

---

### Evidencia 2: Contenedores activos en EC2
Terminal de la instancia EC2 Amazon Linux 2023 mostrando los dos contenedores corriendo: la aplicación `tickets-app` en el puerto 8081 y la base de datos `postgres:16-alpine`.

<img width="1912" height="492" alt="docker ps con contenedores activos" src="https://github.com/user-attachments/assets/015230ec-7d04-468e-9a7c-f8f881d66aee" />

---

### Evidencia 3: Dashboard CloudWatch — métricas de sistema
Dashboard `tickets-app-dashboard` mostrando en tiempo real: uso de CPU (cpu_usage_user + cpu_usage_system), memoria utilizada (58.7%), disco utilizado (48.5%) y tráfico de red (net_bytes_recv + net_bytes_sent).

<img width="1493" height="688" alt="Dashboard CloudWatch métricas de sistema" src="https://github.com/user-attachments/assets/2d66fe89-6a68-4fa2-809a-0aed887e0510" />

---

### Evidencia 4: Dashboard CloudWatch — logs de contenedores Docker
Widget de logs del dashboard mostrando entradas en tiempo real del log group `/tickets-app/docker`, con logs de PostgreSQL y la aplicación Spring Boot capturados por el CloudWatch Agent desde la EC2.

<img width="1885" height="382" alt="Logs de Docker en CloudWatch" src="https://github.com/user-attachments/assets/87a5d08e-f33b-49b1-9515-5dbda50a6d00" />

---

### Evidencia 5: Dashboard CloudWatch — cobertura y tiempo de despliegue
Widgets de texto del dashboard mostrando la cobertura de pruebas reportada por JaCoCo (32% total, con application.usecase al 100%) y el tiempo de despliegue por job del último pipeline exitoso (total: 2m 27s).

<img width="1017" height="433" alt="Widgets de cobertura y tiempo de despliegue" src="https://github.com/user-attachments/assets/c4959790-22e7-4895-a3c6-5e62c914d60d" />

---

### Evidencia 6: SonarCloud — Quality Gate y detección de issues
SonarCloud detectó 3 issues abiertos y el quality gate falló por 1 condición incumplida (vulnerabilidades de seguridad críticas en el código). Esto demuestra que el sistema de cumplimiento funciona y detecta problemas reales de calidad.

<img width="1520" height="443" alt="SonarCloud Quality Gate Failed" src="https://github.com/user-attachments/assets/bbf69e5d-48b8-4b33-964e-ece81d64fb5c" />

---

### Evidencia 7: Branch Protection activa en GitHub
Configuración de protección de rama `main` en GitHub con la regla "Require a pull request before merging" activa. Los push directos a `main` están bloqueados; todo cambio debe ingresar por Pull Request.

<img width="1198" height="625" alt="Branch protection rules en GitHub" src="https://github.com/user-attachments/assets/11583a42-ac01-4dea-baa3-e54a04f9feb3" />

---

## Declaración de Uso de IA

Durante el ciclo de desarrollo de este proyecto, se utilizó el modelo **Gemini** como herramienta de apoyo para la redacción de documentación y asistencia en configuraciones técnicas. No obstante, todas las definiciones arquitectónicas, revisiones de código y decisiones técnicas finales fueron tomadas enteramente por el equipo de desarrollo.

Criterios de uso alineados con: [https://bibliotecas.duoc.cl/ia](https://bibliotecas.duoc.cl/ia)

---

## Reflexión Personal

Honestamente este proyecto me costó más de lo que esperaba. El runner en EC2 me hizo perder horas por cosas tontas, y cuando por fin vi los tres jobs en verde sentí un alivio enorme. Lo que más me sorprendió fue lo útil que resultó CloudWatch, nunca pensé que los logs centralizados me iban a salvar tanto tiempo. Me llevo que el CI/CD no es solo automatización, es tranquilidad.