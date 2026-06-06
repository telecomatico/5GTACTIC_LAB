
FROM free5gc-compose-free5gc-upf:latest

# Instala utilidades mínimas (opcional, útil para copiar y depurar)
RUN apt-get install -y --no-install-recommends \
    bash coreutils procps iproute2 iputils-ping traceroute nano net-tools \
    && rm -rf /var/lib/apt/lists/*

# Copia tus defaults para el seed
COPY config_defaults/ /defaults/

# Declara el volumen persistente
VOLUME ["/free5gc/config"]

# 1) Asegura que exista el directorio del entrypoint
RUN mkdir -p /usr/local/bin

# 2) Crea el entrypoint dentro de la imagen (LF y sin BOM)
RUN cat > /usr/local/bin/docker-entrypoint.sh <<'SH'
#!/bin/sh
set -e

TARGET="/free5gc/config"
DEFAULTS="/defaults"

# Asegura que exista el directorio de destino
[ -d "$TARGET" ] || mkdir -p "$TARGET"

# Si FORCE_SEED=1, sobrescribe; si no, solo si está vacío
if [ "${FORCE_SEED:-0}" = "1" ]; then
  echo "[entrypoint] FORCE_SEED=1: sobrescribiendo $TARGET con defaults..."
  cp -a "$DEFAULTS/." "$TARGET/" 2>/dev/null || cp -r "$DEFAULTS/." "$TARGET/"
else
  if [ -z "$(ls -A "$TARGET" 2>/dev/null)" ]; then
    echo "[entrypoint] Inicializando $TARGET con defaults..."
    cp -a "$DEFAULTS/." "$TARGET/" 2>/dev/null || cp -r "$DEFAULTS/." "$TARGET/"
  else
    echo "[entrypoint] $TARGET ya tiene contenido; no se sobrescribe."
  fi
fi

# Ejecuta el CMD/args que vengan
# Si hay CMD/args, ejecutarlos; si no, entrar en bash
if [ $# -gt 0 ]; then
  exec "$@"
elseelse
  echo "[entrypoint] Sin CMD/args: abriendo /bin/bash -l"
  exec /bin/bash -l
fi
SH

# 3) Aplica permisos y verifica que el archivo existe
RUN chmod 0755 /usr/local/bin/docker-entrypoint.sh && \
    ls -l /usr/local/bin/docker-entrypoint.sh && \
    head -n 3 /usr/local/bin/docker-entrypoint.sh

# 4) Define el entrypoint y deja un CMD de depuración
ENTRYPOINT ["/usr/local/bin/docker-entrypoint.sh"]
CMD ["tail", "-f", "/dev/null"]
