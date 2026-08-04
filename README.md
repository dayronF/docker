# Syncra - Deploy (Docker Compose)

Este repositorio contiene la configuración de Docker para levantar **backend**,
**frontend** y el **reverse proxy (nginx)** juntos, en local o para preparar
el despliegue en producción.

No contiene código de la aplicación — el backend y el frontend viven en sus
propios repositorios, clonados como carpetas hermanas de esta.

## Estructura esperada

```
Syncra/                  <- carpeta contenedora (no es un repo)
├── syncra-deploy/       <- este repo
│   ├── docker-compose.yml
│   ├── nginx.conf
│   └── .env.example
├── Syncra/               <- repo del backend
│   └── .env              <- credenciales del backend (pedir al equipo, NO subir a git)
└── Syncra_front/          <- repo del frontend
```

## Requisitos

- [Docker Desktop](https://www.docker.com/products/docker-desktop/) instalado y **corriendo**
  (el ícono de la ballena debe estar activo, no cargando)
- Los 3 procesos locales (backend con Maven, frontend con `ng serve`) **cerrados**
  antes de levantar Docker, para evitar conflictos de puertos (8080, 4200)

## Setup — primera vez

### 1. Clona los 3 repos como hermanos, respetando estos nombres

```bash
mkdir Syncra
cd Syncra

git clone <url-de-syncra-deploy> syncra-deploy
git clone <url-del-backend> Syncra
git clone <url-del-frontend> Syncra_front
```

> ⚠️ Si prefieres usar otros nombres de carpeta, no pasa nada — solo tienes
> que decírselo a `docker-compose` en el paso 3.

### 2. Pide el `.env` del backend al equipo

El backend necesita un archivo `.env` en su raíz (`Syncra/.env`) con las
credenciales de base de datos, JWT, Gmail y Cloudinary. **No está en git**
por seguridad — pídelo a quien lidera el backend y colócalo en:

```
Syncra/.env
```

### 3. (Opcional) Si tus carpetas se llaman diferente

Si no usaste los nombres exactos del paso 1, copia la plantilla:

```bash
cd syncra-deploy
cp .env.example .env
```

Y edita `.env` con tus rutas reales, por ejemplo:

```dotenv
BACKEND_PATH=../mi-carpeta-backend
FRONTEND_PATH=../mi-carpeta-frontend
```

Este archivo `syncra-deploy/.env` es local tuyo, nunca se sube a git.

### 4. Levanta todo el stack

Desde dentro de `syncra-deploy/`:

```bash
docker-compose up --build
```

La primera vez tarda varios minutos (descarga imágenes base + build de Maven
y npm). Las siguientes veces es más rápido gracias al caché de Docker.

### 5. Prueba

- Abre `http://localhost` en el navegador → debe cargar Angular.
- Abre DevTools → pestaña Network → haz login o cualquier acción que llame
  a la API → confirma que las llamadas a `/api/v1/...` respondan `200`
  (no `404`, no error de CORS, no error de conexión).

## Comandos útiles del día a día

```bash
# Levantar todo (usa caché si no hay cambios de código)
docker-compose up

# Levantar reconstruyendo imágenes (después de cambios en backend/frontend)
docker-compose up --build

# Ver logs de un servicio específico
docker-compose logs backend
docker-compose logs frontend
docker-compose logs nginx

# Ver logs en vivo (streaming)
docker-compose logs -f backend

# Parar todo
docker-compose down

# Ver estado de los contenedores
docker-compose ps
```

## Troubleshooting

| Síntoma                                    | Causa probable                                                                 |
|---------------------------------------------|----------------------------------------------------------------------------------|
| `404` en llamadas a `/api/v1/...`          | Revisa `nginx.conf` — el `proxy_pass` no debe tener `/` al final                |
| Error de CORS en consola del navegador     | Revisa que `SecurityConfig.java` tenga `.cors(cors -> {})` activo                |
| Backend no arranca / error de conexión a BD | Revisa `Syncra/.env` — credenciales o que el trial de Railway no haya expirado  |
| `502 Bad Gateway` en `http://localhost`    | El backend tardó en levantar (Spring Boot toma unos segundos); reintenta        |
| `docker-compose up` no encuentra la carpeta | Revisa que `BACKEND_PATH`/`FRONTEND_PATH` coincidan con tus nombres reales      |
| Cambios de código no se reflejan            | Usa `docker-compose up --build`, no solo `up`                                   |

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
