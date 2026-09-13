# Smart City – Sensor de Temperatura

Plataforma web para gestionar sensores de temperatura, ingerir lecturas desde fuentes heterogéneas y visualizar lecturas e histórico de ingestas.

---

## Índice

1. [Stack tecnológico](#stack-tecnológico)
2. [Estructura del proyecto](#estructura-del-proyecto)
3. [Variables de entorno](#variables-de-entorno)
4. [Ejecución en local](#ejecución-en-local)
5. [Ejecución con Docker](#ejecución-con-docker)
6. [Ejecución con Kubernetes](#ejecución-con-kubernetes)
7. [Decisiones arquitectónicas](#decisiones-arquitectónicas)
8. [Mejoras futuras](#mejoras-futuras)

---

## Stack tecnológico

### Backend
- **NestJS** + TypeScript — framework principal
- **TypeORM** + **PostgreSQL** — persistencia
- **Passport JWT** + cookie httpOnly — autenticación
- **@nestjs/axios** — HTTP polling para sensores externos
- **class-validator** — validación de DTOs

### Frontend
- **React** + TypeScript + **Vite** — aplicación web
- **TanStack Query v5** — gestión de peticiones y caché
- **Tailwind CSS** — estilos responsive mobile-first
- **Recharts** — gráfica de lecturas de temperatura
- **React Router v6** — enrutamiento

### Infraestructura
- **Docker** + Docker Compose — contenedorización
- **Kubernetes** (Minikube) — orquestación
- **Nginx** — servidor del frontend en producción

---

## Estructura del proyecto

```
Sensorization/
├── smartcity-backend/       # API NestJS
│   ├── src/
│   │   ├── auth/            # Login, registro, JWT, guard
│   │   ├── users/           # Entidad y servicio de usuarios
│   │   ├── sensors/         # CRUD de sensores
│   │   ├── ingestions/      # Lógica de ingesta (Formato A y B)
│   │   ├── readings/        # Lecturas de temperatura
│   │   └── mock/            # Endpoints mock para HTTP_POLL
│   ├── Dockerfile
│   └── .env
├── smartcity-frontend/      # App React
│   ├── src/
│   │   ├── pages/           # Login, Sensors, SensorDetail, Ingestions
│   │   ├── hooks/           # useAuth
│   │   ├── lib/             # Cliente axios
│   │   └── components/
│   ├── Dockerfile
│   ├── nginx.conf
│   └── .env
├── k8s/
│   ├── postgres/            # PVC, Deployment, Service
│   ├── backend/             # ConfigMap, Secret, Deployment, Service
│   └── frontend/            # Deployment, Service (NodePort)
└── docker-compose.yml
```

---

## Variables de entorno

### Backend — `smartcity-backend/.env`

| Variable | Descripción | Ejemplo |
|---|---|---|
| `PORT` | Puerto del servidor | `3000` |
| `NODE_ENV` | Entorno de ejecución | `development` |
| `DB_HOST` | Host de PostgreSQL | `localhost` |
| `DB_PORT` | Puerto de PostgreSQL | `5432` |
| `DB_USERNAME` | Usuario de PostgreSQL | `smartcity` |
| `DB_PASSWORD` | Contraseña de PostgreSQL | `smartcity123` |
| `DB_NAME` | Nombre de la base de datos | `smartcity` |
| `JWT_SECRET` | Clave secreta para firmar los JWT | `change_this_in_production` |
| `JWT_EXPIRES_IN` | Tiempo de expiración del token | `7d` |
| `COOKIE_NAME` | Nombre de la cookie de sesión | `auth_token` |
| `FRONTEND_URL` | URL del frontend (para CORS) | `http://localhost:5173` |

### Frontend — `smartcity-frontend/.env`

| Variable | Descripción | Ejemplo |
|---|---|---|
| `VITE_API_URL` | URL base del backend | `http://localhost:3000/api` |

---

## Ejecución en local

### Requisitos previos

- Node.js 20+
- Docker (para PostgreSQL)
- npm

### 1. Base de datos

```bash
docker run --name smartcity-pg \
  -e POSTGRES_USER=smartcity \
  -e POSTGRES_PASSWORD=smartcity123 \
  -e POSTGRES_DB=smartcity \
  -p 5432:5432 -d postgres:16
```

Para arrancarlo en sesiones posteriores:

```bash
docker start smartcity-pg
```

### 2. Backend

```bash
cd smartcity-backend
npm install
npm run start:dev
```

La API estará disponible en `http://localhost:3000/api`.

Las tablas se crean automáticamente al arrancar gracias a `synchronize: true` de TypeORM (solo en `NODE_ENV=development`).

### 3. Frontend

```bash
cd smartcity-frontend
npm install
npm run dev
```

La app estará disponible en `http://localhost:5173`.

---

## Ejecución con Docker

### Requisitos previos

- Docker Desktop

### Levantar todo

Desde la raíz del proyecto:

```bash
docker compose up --build
```

Servicios disponibles:
- Frontend: `http://localhost:5173`
- Backend: `http://localhost:3000/api`

### Reconstruir solo un servicio

```bash
docker compose up --build backend
docker compose up --build frontend
```

### Parar todo

```bash
docker compose down
```

> **Nota:** Los datos de PostgreSQL se persisten en el volumen `postgres_data`. Para borrar también los datos: `docker compose down -v`.

---

## Ejecución con Kubernetes

### Requisitos previos

- Docker Desktop
- Minikube
- kubectl

### 1. Arrancar Minikube

```bash
minikube start
```

### 2. Construir las imágenes dentro de Minikube

```bash
minikube image build -t smartcity-backend:latest ./smartcity-backend
minikube image build -t smartcity-frontend:latest ./smartcity-frontend
```

### 3. Desplegar los manifiestos

```bash
kubectl apply -f k8s/postgres/
kubectl apply -f k8s/backend/
kubectl apply -f k8s/frontend/
```

### 4. Verificar el estado

```bash
kubectl get pods
kubectl get services
```

Todos los pods deben estar en estado `Running`.

### 5. Abrir la aplicación

```bash
minikube service frontend
```

Esto abre el navegador automáticamente con la URL correcta.

### Arquitectura K8s

| Componente | Réplicas | Tipo de Service |
|---|---|---|
| PostgreSQL | 1 | ClusterIP |
| Backend | 2 | ClusterIP |
| Frontend | 2 | NodePort (30000) |

Los datos de PostgreSQL se persisten en un `PersistentVolumeClaim` de 1Gi.

Las variables sensibles (JWT secret, contraseña de BD) se gestionan con `Secret` de Kubernetes. El resto de configuración va en `ConfigMap`.

---

## Decisiones arquitectónicas

### Autenticación con cookie httpOnly
Se eligió cookie httpOnly en lugar de JWT en localStorage por seguridad — la cookie no es accesible desde JavaScript, lo que elimina el riesgo de ataques XSS. El backend setea y limpia la cookie; el frontend nunca toca el token directamente.

### Detección automática de formato de ingesta
El servicio de ingestas detecta el formato del payload automáticamente:
- **Formato A:** array con `sensorCode`, `ts` (ISO 8601) y `valueC`
- **Formato B:** objeto con `deviceId` y array `data[]` con `time` (unix seconds) y `temp`

Si el `sensorCode`/`deviceId` no coincide con el sensor configurado, las lecturas se descartan silenciosamente. Los payloads inválidos generan un `IngestionRun` con `status: error` y mensaje descriptivo.

### Anti-duplicados en lecturas
La entidad `TemperatureReading` tiene un constraint único en `(sensorId, timestamp)`. Si una lectura ya existe, se ignora silenciosamente — el `IngestionRun` refleja solo los registros realmente guardados.

### Módulo Mock para HTTP_POLL
Los endpoints `/api/mock/temp-format-a` y `/api/mock/temp-format-b` simulan fuentes externas de datos. Cuando un sensor es de tipo `HTTP_POLL`, al ingestar la app hace un GET a la URL configurada en el sensor (que puede apuntar a estos mocks o a cualquier fuente real).

### `synchronize: true` solo en desarrollo
TypeORM crea y actualiza las tablas automáticamente en `development`. En producción debería desactivarse y usar migraciones explícitas.

### Nginx como servidor del frontend
El frontend se construye como ficheros estáticos y se sirve con Nginx, con la configuración `try_files` para que React Router funcione al refrescar cualquier ruta.

---

## Mejoras futuras

- **Migraciones TypeORM:** reemplazar `synchronize: true` por migraciones versionadas para entornos de producción reales.
- **Refresh tokens:** implementar tokens de refresco para no expulsar al usuario tras la expiración del JWT.
- **Polling automático:** añadir un cron job con `@nestjs/schedule` para que los sensores `HTTP_POLL` ingesten automáticamente a intervalos configurables.
- **Paginación en la lista de sensores:** cuando haya muchos sensores, añadir paginación o scroll infinito.
- **Tests:** añadir tests unitarios de los servicios de ingesta (parseo de formatos) y tests e2e de los endpoints principales.
- **Ingress en K8s:** reemplazar el NodePort del frontend por un Ingress con un dominio real y TLS.
- **Secretos seguros en K8s:** usar herramientas como Sealed Secrets o HashiCorp Vault en lugar de `stringData` en los manifiestos.
- **Alertas de temperatura:** notificar cuando una lectura supere un umbral configurable por sensor.
- **Roles de usuario:** diferenciar entre administradores (pueden borrar cualquier sensor) y usuarios normales (solo los suyos).