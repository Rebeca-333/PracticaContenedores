# 1. Imagen base oficial y mínima
FROM python:3.12-slim

# Higiene de la imagen: sin .pyc, logs en stdout, pip sin caché
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PIP_NO_CACHE_DIR=1

WORKDIR /app

# 2. Dependencias ANTES del código fuente para aprovechar la caché de capas:
#    solo se reinstala esta capa si cambia requirements.txt
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 3. Usuario del sistema sin privilegios (nunca root)
RUN groupadd --system app \
    && useradd --system --gid app --home-dir /app --shell /usr/sbin/nologin app

# Código fuente propiedad del usuario no privilegiado
COPY --chown=app:app . .

# A partir de aquí todo se ejecuta como 'app'
USER app

# 4. Puerto declarado
EXPOSE 8000

# REDIS_URL no se fija en la imagen: se pasa en tiempo de ejecución
# (docker run -e REDIS_URL=... o docker-compose.yml)

# 5. Comprobación de salud contra /salud usando solo la stdlib de Python
#    (python:3.12-slim no incluye curl ni wget).
#    Si Redis está caído, /salud devuelve 503 -> urlopen lanza excepción
#    -> código de salida != 0 -> contenedor "unhealthy".
HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
    CMD ["python", "-c", "import urllib.request; urllib.request.urlopen('http://127.0.0.1:8000/salud', timeout=3)"]

# 6. Servidor de producción gunicorn (no el servidor de desarrollo de Flask)
#    Formato estricto JSON array (sin shell)
CMD ["gunicorn", "--bind", "0.0.0.0:8000", "--workers", "2", "--threads", "4", "app:app"]
