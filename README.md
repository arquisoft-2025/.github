# 🎁 Proyecto Donatello

**Plataforma de Donaciones Solidarias** — Sistema distribuido basado en microservicios que conecta donadores con personas o comunidades que necesitan recursos. Los donadores publican artículos (ropa, alimentos, muebles, etc.) y los beneficiarios pueden explorarlos, filtrarlos, agregarlos a un carrito y reclamarlos, generando una notificación automática por correo al donador.

> Proyecto desarrollado para el curso de **Arquitectura de Software (2025)** — Organización GitHub: [`arquisoft-2025`](https://github.com/arquisoft-2025)

---

## 📑 Tabla de Contenido

- [Arquitectura General](#-arquitectura-general)
- [Repositorios](#-repositorios)
- [Stack Tecnológico](#-stack-tecnológico)
- [Microservicios](#-microservicios)
  - [API Gateway](#api-gateway-go--fiber)
  - [BE_User_Management](#be_user_management--gestión-de-usuarios)
  - [BE_Donation](#be_donation--publicación-de-donaciones)
  - [BE_Shopping_cart](#be_shopping_cart--carrito)
  - [BE_notification](#be_notification--consulta-y-notificaciones)
  - [Notification-asynchrony](#notification-asynchrony--worker-asíncrono)
- [Frontends](#-frontends)
- [Reverse Proxies y Balanceo de Carga](#-reverse-proxies-y-balanceo-de-carga)
- [Comunicación Asíncrona](#-comunicación-asíncrona-flujo-de-notificaciones)
- [Persistencia](#-persistencia)
- [Seguridad](#-seguridad)
- [Observabilidad](#-observabilidad)
- [Despliegue con Docker Compose](#-despliegue-con-docker-compose)
- [Ejecución en Desarrollo (sin Docker)](#-ejecución-en-desarrollo-sin-docker)
- [Pruebas de Integración](#-pruebas-de-integración)
- [Diagramas de Arquitectura](#-diagramas-de-arquitectura)

---

## 🏗 Arquitectura General

Donatello sigue una **arquitectura de microservicios** con dos canales de entrada independientes (web y móvil), cada uno con su propio **reverse proxy Nginx** que balancea carga entre **3 instancias replicadas del API Gateway**. El Gateway enruta las peticiones a los microservicios de negocio, que persisten en bases de datos aisladas por red y se comunican de forma asíncrona mediante colas de mensajes para el envío de notificaciones.

```
                         ┌──────────────┐        ┌──────────────┐
                         │   FE_Web     │        │  FE_Mobile   │
                         │ (React+Vite) │        │ (RN + Expo)  │
                         └──────┬───────┘        └──────┬───────┘
                                │ HTTPS :8443                │ HTTP :80
                     ┌──────────▼──────────┐     ┌──────────▼──────────┐
                     │  Reverse_Proxy_Web  │     │ Reverse_Proxy_Mobile│
                     │  Nginx + TLS        │     │  Nginx              │
                     │  LB: round_robin    │     │  LB: least_conn     │
                     └──────────┬──────────┘     └──────────┬──────────┘
                                │  pesos [3,2,1]            │  pesos [3,2,1]
                  ┌─────────────┼─────────────┐             │
            ┌─────▼─────┐ ┌─────▼─────┐ ┌─────▼─────┐      ...
            │ Gateway 1 │ │ Gateway 2 │ │ Gateway 3 │   (x3 instancias)
            │ Go+Fiber  │ │ Go+Fiber  │ │ Go+Fiber  │
            └─────┬─────┘ └─────┬─────┘ └─────┬─────┘
                  └─────────────┼─────────────┘
          /api/v1/users  /api/v1/donations  /api/v1/cart  /api/v1/notification
        ┌───────────────┬───────────────┬───────────────┬────────────────┐
┌───────▼───────┐ ┌─────▼─────────┐ ┌───▼───────────┐ ┌─▼──────────────┐
│BE_User_Mgmt   │ │ BE_Donation   │ │BE_Shopping_cart│ │ BE_notification│
│Flask :5002    │ │ Flask :5000   │ │Flask :5003     │ │ Flask :5001    │
└───────┬───────┘ └─────┬─────────┘ └───┬───────────┘ └─┬──────┬───────┘
        │               │               │               │      │
   ┌────▼────┐     ┌────▼────┐     ┌────▼────┐          │ ┌────▼───────────┐
   │  MySQL  │     │ MongoDB │     │ MongoDB │          │ │ RabbitMQ/Redis │
   │(usuarios)│    │(donatio.)│    │ (carts) │          │ │  (mensajería)  │
   └─────────┘     └─────────┘     └─────────┘          │ └────┬───────────┘
                                                        │      │
                                              ┌─────────▼──────▼────────┐
                                              │ Notification-asynchrony │
                                              │  Celery Worker → SMTP   │
                                              └─────────────────────────┘
```

**Decisiones arquitectónicas clave:**

- **Punto de entrada único (API Gateway):** abstrae los microservicios internos y centraliza CORS, logging, manejo de errores y health checks.
- **Alta disponibilidad:** 3 réplicas del Gateway detrás de Nginx con balanceo ponderado, reintentos automáticos (`proxy_next_upstream`) y health checks en cada contenedor.
- **Segmentación de redes Docker:** `frontend-network` (pública), `backend-network`, `database-network` y `messaging-network` (las tres últimas `internal: true`, sin acceso externo).
- **Desacoplamiento asíncrono:** el reclamo de una donación no bloquea al usuario; la notificación por correo se encola (RabbitMQ/Redis) y la procesa un worker Celery con reintentos y backoff exponencial.
- **Base de datos por servicio:** cada microservicio es dueño de su propio almacenamiento (patrón *database-per-service*).

---

## 📦 Repositorios

| Repositorio | Lenguaje | Rol |
|---|---|---|
| [`Api_Gateway`](https://github.com/arquisoft-2025/Api_Gateway) | Go (Fiber) | Punto de entrada único; enruta a los microservicios bajo `/api/v1/*` |
| [`BE_User_Management`](https://github.com/arquisoft-2025/BE_User_Management) | Python (Flask) | Registro, login (JWT) y recuperación de contraseña |
| [`BE_Donation`](https://github.com/arquisoft-2025/BE_Donation) | Python (Flask) | CRUD de donaciones, carga de imágenes, validación por categoría |
| [`BE_Shopping_cart`](https://github.com/arquisoft-2025/BE_Shopping_cart) | Python (Flask) | Carrito y reclamo de donaciones |
| [`BE_notification`](https://github.com/arquisoft-2025/BE_notification) | Python (Flask) | Filtrado de donaciones y disparo de notificaciones a donadores |
| [`Notification-asynchrony`](https://github.com/arquisoft-2025/Notification-asynchrony) | Python (Celery) | Worker asíncrono: envío de correos SMTP y actualización de disponibilidad |
| [`FE_web`](https://github.com/arquisoft-2025/FE_web) | JavaScript (React + Vite) | Aplicación web para donadores y beneficiarios |
| [`FE_Mobile`](https://github.com/arquisoft-2025/FE_Mobile) | TypeScript (React Native + Expo) | Aplicación móvil multiplataforma |
| [`Reverse_Proxy_Web`](https://github.com/arquisoft-2025/Reverse_Proxy_Web) | Nginx / Shell | Proxy inverso HTTPS + balanceador para el canal web; orquesta todo el stack |
| [`Reverse_Proxy_Mobile`](https://github.com/arquisoft-2025/Reverse_Proxy_Mobile) | Nginx / Dockerfile | Proxy inverso + balanceador (`least_conn`) para el canal móvil |
| [`Integration_Tests`](https://github.com/arquisoft-2025/Integration_Tests) | Python | Suite de pruebas de integración (APIs + Selenium) con reportes PDF |

---

## 🛠 Stack Tecnológico

| Capa | Tecnologías |
|---|---|
| **Frontend Web** | React 19, Vite, React Router 7, React Bootstrap, Axios, React Hook Form, tsParticles |
| **Frontend Móvil** | React Native 0.81, Expo ~54, Expo Router, TypeScript 5.9, AsyncStorage |
| **API Gateway** | Go 1.24, Fiber v2 (middlewares: CORS, logger, recover), godotenv |
| **Microservicios** | Python 3.8+, Flask 3, Flasgger (Swagger), Flask-JWT-Extended, Flask-CORS, Flask-Mail |
| **Bases de datos** | MongoDB (donaciones, carrito, usuarios en dev), MySQL 8.0 (usuarios en despliegue) |
| **Mensajería / caché** | RabbitMQ 3 (cola `notification_queue`), Redis 7 (broker/result backend), Celery 5.4 |
| **Infraestructura** | Docker, Docker Compose, Nginx (TLS, gzip, balanceo de carga), certificados autofirmados |
| **Observabilidad** | Prometheus client (métricas en cada API Flask), logs estructurados de Nginx y Fiber |
| **Pruebas** | Requests (integración de APIs), Selenium (UI), generación de reportes PDF |

---

## 🔀 Microservicios

### API Gateway (Go + Fiber)

Punto de entrada único del sistema. Enruta todas las solicitudes hacia los servicios internos y aporta una capa transversal de CORS, logging, recuperación ante pánicos, manejo de errores personalizado y *graceful shutdown*.

**Rutas:**

| Ruta | Servicio destino |
|---|---|
| `GET /health` | Health check del gateway |
| `ALL /api/v1/users/*` | BE_User_Management (`:5002`) |
| `ALL /api/v1/donations/*` | BE_Donation (`:5000`) |
| `ALL /api/v1/cart/*` | BE_Shopping_cart (`:5003`) |
| `ALL /api/v1/notification/*` | BE_notification (`:5001`) |

**Configuración** (`.env`):

```env
PORT=8080
SERVICE_USER_URL=http://127.0.0.1:5002/
SERVICE_DONATION_URL=http://127.0.0.1:5000/
SERVICE_CART_URL=http://127.0.0.1:5003/
SERVICE_NOTIFICATION_URL=http://127.0.0.1:5001/
```

**Ejecución:** `go mod download && go run cmd/app/main.go`

---

### BE_User_Management — Gestión de usuarios

API Flask encargada del registro, autenticación y recuperación de credenciales. Corre en `http://127.0.0.1:5002` con Swagger en `/apidocs`.

- `POST /register` — registro de usuario (contraseña cifrada con **bcrypt**)
- `POST /login` — autenticación; retorna **token JWT**
- Recuperación de contraseña vía **Flask-Mail** con generación de contraseña aleatoria
- Métricas Prometheus: `user_management_http_requests_total`, latencia y errores por endpoint

> Nota: en desarrollo local el servicio usa **MongoDB** (`Flask-PyMongo`); en el despliegue con Docker Compose se provisiona **MySQL 8.0** con script de inicialización.

---

### BE_Donation — Publicación de donaciones

API Flask + MongoDB para que los donadores registren productos. Corre en `http://localhost:5000` con Swagger en `/apidocs`. Estructura modular (`app/routes`, `app/services`, `app/schemas`, `app/utils`).

- `POST /donations` — crear donación (imagen, categoría, ciudad, condición, ubicación)
- `GET /donations` — listar donaciones disponibles
- `GET /donations/all` — listar todas (disponibles e inactivas)
- `PUT /donations/<id>` — alternar disponibilidad
- `DELETE /donations/<id>` — eliminar donación
- Endpoint **GraphQL** (`/graphql`) usado por el worker asíncrono para la mutación `updateDonation` (marca la donación como no disponible al ser reclamada)
- Validación de datos según la categoría del producto y carga de imágenes a `uploads/`

---

### BE_Shopping_cart — Carrito

API Flask + MongoDB en `http://127.0.0.1:5003` (Swagger en `/apidocs`). Permite al beneficiario agregar donaciones a un carrito y reclamarlas. Al confirmar el reclamo, se comunica con:

- `DONATION_API_URL` — para consultar/actualizar la donación
- `NOTIFICATION_BROKER_URL` — para **encolar** la notificación al donador (flujo asíncrono)

Requiere token JWT emitido por el servicio de usuarios.

---

### BE_notification — Consulta y notificaciones

API Flask en `http://127.0.0.1:5001` (Swagger en `/apidocs`). Dos responsabilidades:

1. **Filtrado de donaciones** — `GET /filteredDonations?city=&condition=&category=` (requiere `Authorization: Bearer <token>`)
2. **Notificación a donadores** — endpoint interno `/internal/sendNotification` (protegido con `INTERNAL_SERVICE_KEY`) invocado por el consumidor de la cola; envía correo al donador cuando alguien se interesa en su donación (soporta SMTP/Gmail y Mailjet)

Incluye además un `redis_consumer.py` que consume la cola de RabbitMQ y delega el envío al endpoint interno.

---

### Notification-asynchrony — Worker asíncrono

Worker **Celery** que procesa la tarea `send_notification` de forma desacoplada:

1. Recibe el payload encolado (email del donador, id/título/descripción de la donación, email del interesado, token).
2. Envía el correo por **SMTP** (`smtp.gmail.com:587` por defecto).
3. Ejecuta la mutación **GraphQL** `updateDonation` contra BE_Donation para marcar `availability: false`.

Características: reintentos automáticos (`max_retries=5`) con **backoff exponencial**, límite de 60 s por tarea y seguimiento de estado. Broker configurable: Redis (`redis://redis:6379/0`) o RabbitMQ (`amqp://guest:guest@rabbitmq:5672//`). Incluye también `enqueue_api.py`, un pequeño servicio HTTP (`:5002` interno) que recibe solicitudes de encolado desde el carrito.

---

## 💻 Frontends

### FE_web (React + Vite)

SPA en `http://localhost:5173` con:

- Catálogo de donaciones con filtros por **categoría, ciudad y condición**
- Registro / login / recuperación de contraseña
- Formulario de publicación de donaciones y dashboard del donador
- Carrito de solicitud de artículos y modal de reclamo
- Diseño responsive (Bootstrap) y notificaciones por email

```bash
npm install
npm run dev
```

### FE_Mobile (React Native + Expo)

App móvil multiplataforma (Expo Router, TypeScript) con autenticación, listado y filtrado de donaciones, carrito y reclamo de artículos. Los endpoints de los servicios se configuran vía `.env` (`react-native-dotenv`).

```bash
npm install
npx expo start
```

---

## 🔁 Reverse Proxies y Balanceo de Carga

Ambos canales cuentan con un Nginx como proxy inverso frente a **3 instancias del API Gateway**:

| | Reverse_Proxy_Web | Reverse_Proxy_Mobile |
|---|---|---|
| Algoritmo | `round_robin` (por defecto) | `least_conn` |
| Pesos | 3 / 2 / 1 | 3 / 2 / 1 |
| TLS | ✅ HTTPS `:8443` (TLS 1.2/1.3, certificados autofirmados con `generate_certs.sh`, redirección HTTP→HTTPS, HSTS) | HTTP `:80` |
| Failover | `max_fails=3`, `fail_timeout=30s`, `proxy_next_upstream` con hasta 3 intentos | igual |
| Extras | Gzip, keepalive, `client_max_body_size 50M`, headers de seguridad (`X-Frame-Options`, `nosniff`), endpoints `/proxy/health` y `/proxy/status`, sirve también el frontend React (`location /` → `app:80`) | Gzip, keepalive, mismos endpoints de salud |

---

## 📨 Comunicación Asíncrona (flujo de notificaciones)

```
Beneficiario reclama donación (FE)
        │
        ▼
BE_Shopping_cart ──HTTP──▶ notification-broker-api (enqueue_api.py)
        │                          │ publica mensaje
        │                          ▼
        │                  RabbitMQ (notification_queue)
        │                          │
        │            ┌─────────────┴─────────────┐
        │            ▼                           ▼
        │   notification-consumer         notification-worker (Celery)
        │   → /internal/sendNotification  → SMTP email + backend Redis
        │            │                           │
        │            ▼                           ▼
        │      Email al donador       GraphQL updateDonation
        │                             (availability = false)
        ▼
Respuesta inmediata al usuario (no espera el correo)
```

Este diseño garantiza **desacoplamiento**, **tolerancia a fallos** (reintentos con backoff) y **escalabilidad horizontal** de los workers.

---

## 🗄 Persistencia

| Servicio | Motor | Base de datos | Puerto host (compose) |
|---|---|---|---|
| BE_Donation | MongoDB | `donationsdb` | `27019` |
| BE_Shopping_cart | MongoDB | `shopping_cart_db` | `27019` (compartido) |
| BE_User_Management | MySQL 8.0 | `user_management` | `3309` |
| Celery / caché | Redis 7 | DB 0 (broker) / DB 1 (results) | `6381` |
| Mensajería | RabbitMQ 3 (+ UI de gestión) | `notification_queue` | `5674` / UI `15674` |

Todos los datos se persisten en **volúmenes Docker** (`web_mongo_data`, `web_mysql_data`, etc.).

---

## 🔐 Seguridad

- **Autenticación JWT** (Flask-JWT-Extended) compartida entre servicios mediante `JWT_SECRET_KEY` común
- Contraseñas cifradas con **bcrypt**
- **TLS** en el proxy web y en los gateways (`ENABLE_HTTPS=true`)
- Endpoint interno de notificaciones protegido con `INTERNAL_SERVICE_KEY`
- Redes Docker internas (`internal: true`) que impiden el acceso externo directo a backends, bases de datos y brokers
- Headers de seguridad en Nginx: `Strict-Transport-Security`, `X-Content-Type-Options`, `X-Frame-Options`
- CORS restringido/configurado tanto en el Gateway como en cada API

> ⚠️ Los valores de `JWT_SECRET_KEY` y credenciales presentes en los `docker-compose.yml` son de **desarrollo**; en producción deben inyectarse mediante variables de entorno o un gestor de secretos.

---

## 📈 Observabilidad

- Cada API Flask expone **métricas Prometheus** (`prometheus_client`): contadores de requests, histogramas de latencia y contadores de errores por endpoint
- Health checks en todos los contenedores (gateways, APIs, MongoDB, MySQL, proxy)
- Logs de acceso/errores de Nginx persistidos en volumen (`reverse-proxy-logs`)
- Logging estructurado en el Gateway (Fiber logger: `[time] status - method path latency`)

---

## 🚀 Despliegue con Docker Compose

El `docker-compose.yml` de **`Reverse_Proxy_Web`** orquesta el stack completo del canal web (proxy, 3 gateways, 4 APIs, broker/worker/consumer de notificaciones, MongoDB, MySQL, Redis y RabbitMQ). El de **`Reverse_Proxy_Mobile`** hace lo propio para el canal móvil.

**Requisito:** clonar todos los repositorios como hermanos en un mismo directorio, ya que los `build.context` usan rutas relativas (`../Api_Gateway`, `../BE_Donation`, ...).

```bash
mkdir donatello && cd donatello
for r in Api_Gateway BE_Donation BE_Shopping_cart BE_User_Management \
         BE_notification FE_web FE_Mobile Notification-asynchrony \
         Reverse_Proxy_Web Reverse_Proxy_Mobile Integration_Tests; do
  git clone https://github.com/arquisoft-2025/$r
done

# Canal web
cd Reverse_Proxy_Web
./generate_certs.sh          # certificados autofirmados para TLS
docker compose up --build -d
```

**Variables de entorno esperadas** (correo): `MAIL_USERNAME`, `MAIL_PASSWORD`, `MAIL_SERVER`, `MAIL_PORT`, `MAIL_SENDER`, `MAILJET_API_KEY_PUBLIC`, `MAILJET_API_KEY_PRIVATE`, `INTERNAL_SERVICE_KEY`.

**Puntos de acceso tras el despliegue (canal web):**

| Recurso | URL |
|---|---|
| Aplicación (vía proxy, HTTPS) | `https://localhost:8443` |
| APIs (vía proxy) | `https://localhost:8443/api/v1/...` |
| Gateways directos | `:8080`, `:8084`, `:8085` |
| APIs directas (debug) | Donaciones `:5010`, Notificaciones `:5011`, Usuarios `:5012`, Carrito `:5013` |
| RabbitMQ Management | `http://localhost:15674` |
| Salud del proxy | `https://localhost:8443/proxy/health` y `/proxy/status` |

---

## 🧪 Ejecución en Desarrollo (sin Docker)

Cada backend Flask sigue el mismo patrón:

```bash
python -m venv venv
source venv/bin/activate      # Windows: .\venv\Scripts\activate
pip install -r requirements.txt
python app.py
```

| Servicio | URL | Swagger |
|---|---|---|
| BE_Donation | `http://127.0.0.1:5000` | `/apidocs` |
| BE_notification | `http://127.0.0.1:5001` | `/apidocs` |
| BE_User_Management | `http://127.0.0.1:5002` | `/apidocs` |
| BE_Shopping_cart | `http://127.0.0.1:5003` | `/apidocs` |
| Api_Gateway | `http://127.0.0.1:8080` | — |
| FE_web | `http://localhost:5173` | — |

Prerrequisitos locales: MongoDB, Redis (para el flujo asíncrono), Node.js ≥16 y Go ≥1.24.

---

## ✅ Pruebas de Integración

El repositorio **`Integration_Tests`** valida el sistema de extremo a extremo:

- **Backend** (`backend/`): flujos de usuario (registro/login/perfil), donaciones, notificaciones y carrito, orquestados por `main_backend_tests.py`
- **Frontend** (`frontend/`): pruebas de UI con **Selenium** sobre `http://localhost:5173` (`frontend_tests.py`)
- **Reportes**: al finalizar cada suite se genera automáticamente un **PDF** con los resultados en `backend/reports/` y `frontend/reports/`

```bash
cd Integration_Tests
pip install -r requirements.txt
python backend/main_backend_tests.py     # requiere backend en ejecución
python frontend/frontend_tests.py        # requiere FE_web en ejecución
```

---

## 📐 Diagramas de Arquitectura

Las vistas arquitectónicas completas del sistema (contexto, componentes, despliegue) están disponibles en Lucidchart:

🔗 [Vistas de arquitectura — Lucidchart](https://lucid.app/lucidchart/4a4be31a-fde7-4e1d-a5ac-0ae9b09deac5/edit?viewport_loc=2%2C-28%2C1609%2C901%2C-Zb.evUI7P0d&invitationId=inv_dc7cc941-80ae-4e57-8da6-ac39149b29fb)

---

## 👥 Equipo

Proyecto desarrollado por el equipo **Donatello** — Arquitectura de Software, 2025.

## 📄 Licencia

Proyecto académico. Consulta con los autores antes de reutilizar el código.
