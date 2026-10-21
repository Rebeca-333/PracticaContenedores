# Tablón de notas — Flask + Redis con Docker Compose

Aplicación Flask (servida con Gunicorn) y caché Redis 7 orquestadas con Docker
Compose. La imagen de la aplicación se define en `notas/Dockerfile` y el
orquestador en `notas/docker-compose.yml`.

> **Nota:** todos los comandos de este documento se ejecutan desde el directorio
> `notas/`, que es donde está el archivo `docker-compose.yml`:
>
> ```bash
> cd notas
> ```

## 1. Requisitos previos

- **Docker Engine** 20.10 o superior (con el plugin de Compose instalado).
- **Docker Compose v2** (el subcomando `docker compose`, sin guion).

Comprobación de ambas piezas:

```bash
docker --version
docker compose version
```

En Debian/Ubuntu el plugin de Compose v2 viene en el paquete
`docker-compose-plugin`. No hace falta el binario antiguo `docker-compose` (v1).

## 2. Variables de entorno

Se declaran en `notas/docker-compose.yml`, dentro del servicio `web`, en la
sección `environment:`:

| Variable     | Valor en el compose       | Qué controla                                                        |
|--------------|---------------------------|---------------------------------------------------------------------|
| `REDIS_URL`  | `redis://cache:6379/0`    | Dónde está Redis: el servicio `cache`, puerto 6379, base de datos 0 |
| `TITULO_APP` | `Tablón de notas`         | Título que se muestra en la página                                   |

También podrían definirse fuera de la imagen con un archivo `.env`
(`docker compose --env-file .env up -d`) o con `docker run -e NOMBRE=valor`.

**¿Por qué no van dentro de la imagen?**

- **Portabilidad:** la misma imagen sirve para desarrollo, pruebas y
  producción; solo cambia la configuración, no el programa.
- **Sin reconstrucciones:** cambiar un valor no obliga a volver a construir la
  imagen; basta volver a levantar el entorno.
- **Seguridad:** las imágenes se exportan y se comparten; cualquier valor
  incrustado en una capa sería legible con `docker history` o
  `docker inspect`, incluidos secretos.
- El Dockerfile solo fija valores de higiene del propio proceso
  (`PYTHONDONTWRITEBYTECODE`, `PYTHONUNBUFFERED`, `PIP_NO_CACHE_DIR`), que sí
  conviene llevar dentro.

Además, `app.py` lee ambas variables con `os.environ.get(...)` y valores por
defecto, de modo que la aplicación también funciona fuera de Docker.

## 3. Puesta en marcha

Desde el directorio `notas/`, el comando exacto para levantar el entorno en
segundo plano y construir las imágenes es:

```bash
docker compose up -d --build
```

- `-d` → ejecución en segundo plano (daemon); el terminal queda libre.
- `--build` → fuerza la reconstrucción de la imagen `web` si el Dockerfile o
  `requirements.txt` han cambiado.

El servicio `web` no arranca hasta que `cache` supera su healthcheck
(`redis-cli ping` → `PONG`). Estado general del despliegue:

```bash
docker compose ps
```

Ambos servicios deben aparecer como `running` (y `healthy` en el caso de
`cache`).

## 4. Pruebas de comprobación

Comprobar que ambos contenedores están `healthy`:

```bash
docker compose ps
```

Endpoint de salud (debe responder HTTP 200 con `{"estado": "ok", ...}`; si
Redis estuviera caído devolvería 503 con `{"estado": "sin cache"}`):

```bash
curl -i http://localhost:8000/salud
```

Portada (debe responder HTTP 200 con HTML):

```bash
curl -i http://localhost:8000/
```

Crear una nota (POST que responde con una redirección a la portada):

```bash
curl -i -X POST -d "texto=Primera nota" http://localhost:8000/notas
```

Ver la nota guardada en el listado:

```bash
curl -s http://localhost:8000/ | grep "Primera nota"
```

Desde el navegador: abrir <http://localhost:8000/>, escribir una nota en el
formulario, enviarla y recargar la página para confirmar que persiste.

## 5. Persistencia de datos

Redis escribe en el volumen nombrado `notas-redis-data`, montado en `/data`,
con persistencia AOF activada (`redis-server --appendonly yes`). Así los datos
sobreviven a reinicios de los contenedores y a un `docker compose down`.

No hay ningún directorio del proyecto implicado: el volumen lo gestiona Docker
dentro de su propio directorio de volúmenes. Para ver la ruta exacta en el host:

```bash
docker volume inspect notas-redis-data
```

En una instalación estándar de Docker en Linux suele ser
`/var/lib/docker/volumes/notas-redis-data/_data` (en modo rootless,
`~/.local/share/docker/volumes/notas-redis-data/_data`). Para inspeccionar los
datos directamente:

```bash
ls -l /var/lib/docker/volumes/notas-redis-data/_data
```

(con permisos de root, `sudo ls -l ...`).

## 6. Visualización de registros

Ver los logs de todos los servicios en vivo (`Ctrl+C` solo corta el seguimiento
en terminal, no detiene los contenedores):

```bash
docker compose logs -f
```

De un servicio concreto y con un número limitado de líneas:

```bash
docker compose logs --tail=100 -f web
```

## 7. Parada y limpieza

Detener los servicios conservando imágenes, volúmenes y datos:

```bash
docker compose down
```

Eliminar por completo contenedores, imágenes y volúmenes generados por el
proyecto:

```bash
docker compose down --rmi all --volumes --remove-orphans
```

Tras este último comando hay que volver a ejecutar
`docker compose up -d --build` para levantar el entorno de nuevo (los datos de
Redis guardados en el volumen se habrán perdido).
