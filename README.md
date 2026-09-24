# Syncra - Deploy (Docker Compose)

Este repositorio contiene la configuración de Docker para levantar **backend**,
**frontend**, el **servicio de IA (ai-service)** y el **reverse proxy (nginx)**
juntos, en local o para preparar el despliegue en producción.

No contiene código de la aplicación — el backend, el frontend y el servicio de
IA viven en sus propios repositorios, clonados como carpetas hermanas de esta.

## Los 4 repositorios

| Carpeta       | Repositorio                                             | Rama en uso                        |
|---------------|----------------------------------------------------------|-------------------------------------|
| `docker/`     | github.com/dayronF/docker                                | `develop`                            |
| `Syncra/`     | github.com/rodriguezsmith2008-web/Syncra                 | `feature/upgradeEditorAndDashboard`  |
| `Syncra_front/` | github.com/rodriguezsmith2008-web/Syncra_front         | `feature/upgradeEditorAndDashboard`  |
| `ai-service/` | github.com/iRay1h/ai-service                             | `feature/AI`                         |

> Quien vaya a desplegar necesita acceso (o que los repos sean públicos) a los
> **4**, y debe clonar la rama indicada arriba — no la rama por defecto — para
> tener las últimas correcciones.

## Estructura esperada

```
Syncra/                  <- carpeta contenedora (no es un repo)
├── docker/              <- este repo
│   ├── docker-compose.yml
│   ├── nginx.conf
│   └── .env.example
├── Syncra/               <- repo del backend
│   └── .env              <- credenciales del backend (pedir al equipo, NO subir a git)
├── Syncra_front/          <- repo del frontend
└── ai-service/            <- repo del servicio de IA
    └── .env               <- credenciales de OpenRouter (pedir al equipo, NO subir a git)
```

## Requisitos

- [Docker Desktop](https://www.docker.com/products/docker-desktop/) instalado y **corriendo**
  (el ícono de la ballena debe estar activo, no cargando)
- Puerto **80** libre en la máquina donde se despliega (es el único puerto que
  se publica al host; backend, frontend e IA solo son visibles entre
  contenedores)
- Los procesos locales (backend con Maven, frontend con `ng serve`, IA con
  uvicorn) **cerrados** antes de levantar Docker, para evitar conflictos de
  puertos si vas a probar en la misma máquina donde desarrollas

## Setup — primera vez

### 1. Clona los 4 repos como hermanos, respetando estos nombres

```bash
mkdir Syncra
cd Syncra

git clone -b develop <url-de-docker> docker
git clone -b feature/upgradeEditorAndDashboard <url-del-backend> Syncra
git clone -b feature/upgradeEditorAndDashboard <url-del-frontend> Syncra_front
git clone -b feature/AI <url-del-ai-service> ai-service
```

> ⚠️ Si prefieres usar otros nombres de carpeta, no pasa nada — solo tienes
> que decírselo a `docker compose` con las variables `BACKEND_PATH`,
> `FRONTEND_PATH` y `AI_SERVICE_PATH` (ver paso 3).

### 2. Pide los archivos `.env` al equipo

Dos servicios necesitan credenciales que **no están en git** por seguridad:

```
Syncra/.env       <- base de datos, JWT, Gmail, Cloudinary
ai-service/.env   <- API key de OpenRouter
```

Pídelos a quien lidera cada parte y colócalos exactamente en esas rutas.

### 3. (Opcional) Si tus carpetas se llaman diferente

Copia la plantilla y ajusta las rutas:

```bash
cd docker
cp .env.example .env
```

```dotenv
BACKEND_PATH=../mi-carpeta-backend
FRONTEND_PATH=../mi-carpeta-frontend
AI_SERVICE_PATH=../mi-carpeta-ia
```

Este archivo `docker/.env` es local tuyo, nunca se sube a git.

### 4. Levanta todo el stack

Desde dentro de `docker/`:

```bash
docker compose -f docker-compose.yml up -d --build
```

La primera vez tarda varios minutos (descarga imágenes base + build de Maven
y npm). Las siguientes veces es más rápido gracias al caché de Docker.

### 5. Prueba

- Abre `http://localhost` en el navegador → debe cargar Angular.
- Abre DevTools → pestaña Network → haz login o cualquier acción que llame
  a la API → confirma que las llamadas a `/api/v1/...` respondan `200`
  (no `404`, no error de CORS, no error de conexión).
- `docker compose -f docker-compose.yml ps` → los 4 servicios deben quedar
  `healthy` (nginx no tiene healthcheck propio, pero depende de que los otros
  3 lo estén para arrancar).

## Comandos útiles del día a día

```bash
# Levantar todo en segundo plano (usa caché si no hay cambios de código)
docker compose -f docker-compose.yml up -d

# Levantar reconstruyendo imágenes (después de cambios en backend/frontend/ia)
docker compose -f docker-compose.yml up -d --build

# Ver logs de un servicio específico
docker compose -f docker-compose.yml logs backend
docker compose -f docker-compose.yml logs frontend
docker compose -f docker-compose.yml logs ai-service
docker compose -f docker-compose.yml logs nginx

# Ver logs en vivo (streaming)
docker compose -f docker-compose.yml logs -f backend

# Parar todo
docker compose -f docker-compose.yml down

# Ver estado de los contenedores
docker compose -f docker-compose.yml ps
```

Para desarrollo local con hot-reload (compila en vivo desde el código fuente
montado, más lento en el primer arranque) usa `docker-compose.dev.yml` en su
lugar — mismos comandos, cambiando el nombre del archivo. Los dos archivos
tienen nombres de proyecto distintos (`syncra-prod` / `syncra-dev`), así que
puedes tener ambos construidos sin que se pisen las imágenes.

## Troubleshooting

| Síntoma                                    | Causa probable                                                                 |
|---------------------------------------------|----------------------------------------------------------------------------------|
| `404` en llamadas a `/api/v1/...`          | Revisa `nginx.conf` — el `proxy_pass` no debe tener `/` al final                |
| Error de CORS en consola del navegador     | Revisa que `SecurityConfig.java` tenga `.cors(cors -> {})` activo                |
| Backend no arranca / error de conexión a BD | Revisa `Syncra/.env` — credenciales o que el trial de Railway no haya expirado  |
| `502 Bad Gateway` en `http://localhost`    | El backend tardó en levantar (Spring Boot toma unos segundos); reintenta        |
| `docker compose up` no encuentra la carpeta | Revisa que `BACKEND_PATH`/`FRONTEND_PATH`/`AI_SERVICE_PATH` coincidan con tus nombres reales |
| Cambios de código no se reflejan            | Usa `docker compose -f docker-compose.yml up -d --build`, no solo `up`          |
| `docker-compose.dev.yml` tarda en verse `healthy` | Normal la primera vez: `spring-boot:run` recompila todo desde cero (2-3 min en Windows) |
| Backend/frontend corren el comando equivocado (p. ej. `jarfile app.jar` no encontrado en dev, o `npx: not found`) | Las imágenes de prod y dev quedaron con el mismo nombre; con el `name:` que ya tiene cada compose no debería repetirse, pero si pasa: `docker compose -f docker-compose.dev.yml build` fuerza reconstruir con el Dockerfile correcto |

## Notas de arquitectura

- El backend usa `context-path: /api/v1/` (ver `application.yml`), por eso
  el `proxy_pass` en `nginx.conf` **no** lleva `/` al final: así conserva el
  prefijo completo al reenviar la petición.
- `frontend` y `backend` no exponen puertos al host (`expose`, no `ports`) —
  solo `nginx` lo hace, porque es el único punto de entrada público.
  Dentro de la red de Docker Compose, `backend` y `frontend` son hostnames
  resolubles automáticamente entre contenedores.
- El Angular (`environment.prod.ts`) usa `apiUrl: '/api/v1'` (ruta relativa),
  no una URL absoluta — así el mismo build funciona sin importar el dominio
  donde se despliegue.
- Los 4 servicios tienen `restart: unless-stopped`: si el host se reinicia o
  un contenedor se cae, vuelven a levantarse solos sin intervención manual.
- **No hay HTTPS configurado** — `nginx.conf` solo escucha en el puerto 80.
  Si el despliegue va a quedar expuesto públicamente con un dominio propio,
  hay que agregar un certificado (por ejemplo con Certbot/Let's Encrypt)
  antes de considerar el despliegue "listo para producción real".
