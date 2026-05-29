# Backend Ventas y Despachos

Dos microservicios REST desarrollados en Spring Boot para gestionar ventas y despachos. Cada servicio se conecta a una base de datos MySQL y se ejecuta en contenedores Docker.

## Microservicios

**back-ventas** — puerto 8080
- Gestiona órdenes de compra
- Endpoints: `GET /api/v1/ventas`, `POST /api/v1/ventas`, `PUT /api/v1/ventas/{id}`, `DELETE /api/v1/ventas/{id}`

**back-despachos** — puerto 8081
- Gestiona órdenes de despacho
- Endpoints: `GET /api/v1/despachos`, `POST /api/v1/despachos`, `PUT /api/v1/despachos/{id}`

## Tecnologías

- Java 17 + Spring Boot 3.4.4
- MySQL 8.0
- Docker multi-stage (Maven → JRE Alpine)
- GitHub Actions
- AWS EC2

## Ejecución local

Requisitos: Docker Desktop instalado.

Crear archivo `.env` basado en `.env.example`:

```
MYSQL_ROOT_PASSWORD=Admin1234
MYSQL_DATABASE=innovatech
MYSQL_USER=appuser
MYSQL_PASSWORD=App1234
DOCKERHUB_USERNAME=tu_usuario
```

Levantar el stack completo:

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
| `DB_ENDPOINT` | Host de la base de datos |
| `DB_PORT` | Puerto MySQL (3306) |
| `DB_NAME` | Nombre de la base de datos |
| `DB_USERNAME` | Usuario de la base de datos |
| `DB_PASSWORD` | Contraseña de la base de datos |

## Persistencia

Se usa un named volume `mysql_data` para que los datos de MySQL no se pierdan al reiniciar contenedores.

## Pipeline CI/CD

El workflow `.github/workflows/ci-cd.yml` se activa con push a la rama `deploy`:

1. Construye las imágenes Docker de ambos backends
2. Publica las imágenes en Docker Hub con tags `latest` y `sha`
3. Copia `docker-compose.prod.yml` a la instancia EC2
4. Conecta por SSH y levanta los contenedores con las nuevas imágenes

## Despliegue en EC2

La instancia EC2 backend debe tener Docker instalado. El pipeline maneja el despliegue automáticamente al hacer push a `deploy`.

Secrets requeridos en GitHub:
- `DOCKERHUB_USERNAME` / `DOCKERHUB_TOKEN`
- `EC2_BACKEND_HOST` / `EC2_USER` / `EC2_SSH_KEY`
- `MYSQL_ROOT_PASSWORD` / `MYSQL_DATABASE` / `MYSQL_USER` / `MYSQL_PASSWORD`
