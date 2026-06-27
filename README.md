# Backend Ventas y Despachos

Dos microservicios REST desarrollados en Spring Boot para gestionar ventas y despachos. Cada servicio corre en AWS ECS Fargate, conectado a una base de datos MySQL gestionada en AWS RDS.

## Microservicios

**back-ventas** — puerto 8080
- Gestiona órdenes de compra
- Endpoints: `GET /api/v1/ventas`, `POST /api/v1/ventas`, `PUT /api/v1/ventas/{id}`, `DELETE /api/v1/ventas/{id}`

**back-despachos** — puerto 8081
- Gestiona órdenes de despacho
- Endpoints: `GET /api/v1/despachos`, `POST /api/v1/despachos`, `PUT /api/v1/despachos/{id}`

## Tecnologías

- Java 17 + Spring Boot 3.4.4
- MySQL 8.0 (AWS RDS)
- Docker multi-stage (Maven → JRE Alpine)
- GitHub Actions (build → push ECR → deploy ECS)
- AWS ECS Fargate + ECR + ALB interno + CloudWatch Logs

## Arquitectura de despliegue

Cada microservicio corre como un servicio ECS Fargate independiente, detrás de un **ALB interno** (no expuesto a Internet) con un listener por puerto:

- ALB interno `:8080` → Target Group `tg-back-ventas` → `svc-back-ventas`
- ALB interno `:8081` → Target Group `tg-back-despachos` → `svc-back-despachos`

El frontend (otro repo) accede a ambos a través del DNS de ese ALB interno. Ningún backend tiene IP pública alcanzable directamente desde Internet — solo el ALB interno y, detrás de él, las tasks Fargate (Security Groups encadenados: ALB público → frontend → ALB interno → backend → RDS).

Scripts de aprovisionamiento de toda la infraestructura (VPC, SGs, RDS, ALBs, cluster, servicios, autoscaling): ver `../aws-setup/` en la raíz del proyecto. Task definitions base: `ecs/back-ventas-taskdef.json`, `ecs/back-despachos-taskdef.json`.

## Ejecución local

Requisitos: Docker Desktop instalado.

Crear archivo `.env` basado en `.env.example`:

```
MYSQL_ROOT_PASSWORD=Admin1234
MYSQL_DATABASE=innovatech
MYSQL_USER=appuser
MYSQL_PASSWORD=App1234
```

Levantar el stack completo (MySQL containerizado, solo para desarrollo local — en producción se usa RDS):

```bash
docker compose up --build
```

Servicios disponibles:
- `http://localhost:8080/api/v1/ventas`
- `http://localhost:8081/api/v1/despachos`
- Swagger UI: `http://localhost:8080/swagger-ui.html`

## Dockerfile

Ambos servicios usan build multi-stage:
- Etapa 1: compila el JAR con Maven 3.9 + JDK 17 Alpine
- Etapa 2: ejecuta el JAR con JRE 17 Alpine (imagen mínima)

El contenedor corre con usuario no root por seguridad.

## Variables de entorno

| Variable | Descripción |
|----------|-------------|
| `DB_ENDPOINT` | Endpoint de RDS en producción (`mysql` en local) |
| `DB_PORT` | Puerto MySQL (3306) |
| `DB_NAME` | Nombre de la base de datos (`innovatech`) |
| `DB_USERNAME` | Usuario de la base de datos |
| `DB_PASSWORD` | Contraseña de la base de datos |

En ECS, estas variables se definen en la `environment` de cada task definition (`ecs/*.json`), apuntando al endpoint real de RDS.

## Autoscaling

Application Auto Scaling con Target Tracking al **50% de uso de CPU** (min 1 / max 3 tasks) en los 3 servicios (ventas, despachos y frontend). Justificación: deja margen de reacción antes de saturar la task, sin disparar scale-out por picos cortos de CPU.

## Pipeline CI/CD

El workflow `.github/workflows/ci-cd.yml` se activa con push a la rama `deploy`, con dos jobs independientes (ventas y despachos) que corren en paralelo:

1. Build de la imagen Docker del servicio correspondiente
2. Login en Amazon ECR y push con tags `latest` y `${{ github.sha }}`
3. Descarga la task definition actual desde ECS
4. Renderiza una nueva revisión con la imagen recién publicada
5. Despliega la nueva revisión en el servicio ECS correspondiente y espera a que quede estable

Secrets requeridos en GitHub:
- `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` / `AWS_SESSION_TOKEN` — credenciales de la sesión de AWS Academy Learner Lab. **Expiran**: hay que refrescarlas en GitHub Secrets antes de cada sesión de trabajo o demo.

## Logs y troubleshooting

```bash
aws logs tail /ecs/back-ventas --follow --region us-east-1
aws logs tail /ecs/back-despachos --follow --region us-east-1
aws ecs describe-services --cluster cluster-innovatech --services svc-back-ventas svc-back-despachos --region us-east-1
```

Si el pipeline falla en el paso de credenciales AWS (`ExpiredToken`/`InvalidClientTokenId`), la sesión del Lab venció — refresca los 3 secrets y vuelve a correr el workflow.
