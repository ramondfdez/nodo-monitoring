# NODO

## Edge Control para Docker

NODO es un panel ligero de observabilidad y operación para equipos que ejecutan Docker. Presenta en una sola interfaz el estado del nodo, métricas de rendimiento, contenedores, procesos y logs.

Está diseñado para funcionar con el mismo `docker-compose.yml` en Docker Desktop para macOS, Linux y dispositivos ARM como OrangePi.

## Qué incluye

- Dashboard operativo con CPU, memoria, disco, red, temperatura y uptime.
- Histórico de CPU y memoria con persistencia en un volumen Docker.
- Listado de contenedores activos y detenidos.
- Búsqueda y filtros por estado.
- Acciones para iniciar, detener, reiniciar y eliminar contenedores.
- Consulta de los últimos logs de cada contenedor.
- Tabla de procesos ordenada por consumo de CPU.
- Actualización automática cada 5 segundos.
- Interfaz responsive para escritorio, tablet y móvil.
- Proxy interno de API mediante Nginx.

## Arquitectura

```text
Navegador
    |
    | HTTP : WEB_PORT
    v
Nginx / Frontend  ------ /api ------>  Backend Go
                                         |
                                         +--> Docker Engine
                                         +--> Métricas del host (opcional)
                                         +--> Histórico en volumen Docker
```

| Componente | Tecnología | Función |
| --- | --- | --- |
| Frontend | HTML, CSS y JavaScript | Interfaz responsive y gráficas |
| Servidor web | Nginx | Archivos estáticos y proxy `/api` |
| Backend | Go + Gorilla Mux | API REST y operaciones Docker |
| Persistencia | Volumen Docker | Histórico de métricas |

## Requisitos

- Docker Engine y Compose, o Docker Desktop en macOS.
- Acceso al socket `/var/run/docker.sock`.
- 512 MB de RAM disponibles como referencia mínima.
- Red local entre el equipo que ejecuta NODO y el móvil o navegador cliente.

## Inicio rápido

### macOS, Linux o Docker Desktop

La configuración base es portable y mantiene desactivadas las acciones de mantenimiento del sistema anfitrión.

```bash
git clone <tu-repositorio> nodo
cd nodo
WEB_PORT=8080 docker-compose up -d --build
```

Abre:

```text
http://localhost:8080
```

Desde otro dispositivo de la misma red, sustituye `localhost` por la IP del equipo que ejecuta Docker:

```text
http://IP_DEL_EQUIPO:8080
```

### OrangePi Linux

En Linux puedes habilitar métricas reales del host y las acciones de mantenimiento usando el mismo Compose:

```bash
WEB_PORT=80 \
HOST_PROC_SOURCE=/proc \
HOST_SYS_SOURCE=/sys \
HOST_ETC_SOURCE=/etc \
NODO_PRIVILEGED=true \
ENABLE_SYSTEM_ACTIONS=true \
docker-compose up -d --build
```

Después, accede desde el móvil o cualquier equipo de la red:

```text
http://IP_DE_LA_ORANGEPI
```

> En instalaciones nuevas, `docker compose` es el comando moderno equivalente a `docker-compose`.

## Variables de configuración

Todas las variables son opcionales. La configuración base funciona sin definirlas.

| Variable | Valor por defecto | Uso |
| --- | --- | --- |
| `WEB_PORT` | `8080` | Puerto público del frontend Nginx |
| `HOST_PROC_SOURCE` | `/tmp` | Directorio del host que se monta como `/host/proc` |
| `HOST_SYS_SOURCE` | `/tmp` | Directorio del host que se monta como `/host/sys` |
| `HOST_ETC_SOURCE` | `/tmp` | Directorio del host que se monta como `/host/etc` |
| `NODO_PRIVILEGED` | `false` | Habilita privilegios adicionales del backend |
| `ENABLE_SYSTEM_ACTIONS` | `false` | Permite reiniciar o actualizar el sistema anfitrión |

Las tres variables `HOST_*_SOURCE`, junto con `NODO_PRIVILEGED=true` y `ENABLE_SYSTEM_ACTIONS=true`, solo deben activarse en un host Linux controlado, como la OrangePi.

El backend no se publica directamente en el host. Nginx recibe las peticiones del navegador y reenvía internamente las rutas `/api` al servicio Go.

## Acceso remoto

Para acceder desde fuera de la red local, utiliza una red privada como:

- Tailscale.
- WireGuard.
- Una VPN equivalente administrada por ti.

No publiques directamente el puerto de NODO en Internet. El backend tiene acceso operativo al socket de Docker y debe tratarse como una consola de administración, no como una web pública.

Si necesitas exponerlo públicamente, añade como mínimo:

1. HTTPS con certificados válidos.
2. Autenticación delante de Nginx.
3. Firewall con allowlist de IPs.
4. Registro y monitorización de accesos.

## Operaciones habituales

### Ver estado de los servicios

```bash
docker-compose ps
```

### Ver logs

```bash
docker-compose logs -f backend
docker-compose logs -f frontend
```

### Reiniciar la aplicación

```bash
docker-compose restart
```

### Actualizar después de cambios

```bash
docker-compose down
docker-compose up -d --build
```

### Detener la aplicación

```bash
docker-compose down
```

### Ver consumo de NODO

```bash
docker stats nodo-backend nodo-frontend
```

## Seguridad y permisos

NODO monta `/var/run/docker.sock` para poder consultar y operar contenedores. Este socket equivale, en la práctica, a un nivel elevado de control sobre Docker.

Recomendaciones:

- Mantener NODO dentro de una red privada.
- No publicar el backend directamente.
- No activar `NODO_PRIVILEGED` ni `ENABLE_SYSTEM_ACTIONS` en macOS o entornos no controlados.
- Proteger el acceso remoto mediante VPN.
- Añadir autenticación y HTTPS antes de cualquier exposición pública.

## Solución de problemas

### La página no carga

Comprueba que el frontend está activo y que el puerto publicado coincide con `WEB_PORT`:

```bash
docker-compose ps
curl http://localhost:8080
```

Si usaste otro puerto, sustituye `8080` en la URL.

### La API no responde

Revisa los logs del backend y confirma que ambos servicios comparten la red Compose:

```bash
docker-compose logs backend
docker-compose config
```

### No aparecen contenedores

Comprueba el socket de Docker y el acceso desde el backend:

```bash
docker inspect nodo-backend | grep docker.sock
docker ps -a
```

### Las métricas del host aparecen vacías

En macOS y en el modo base son esperables métricas limitadas del host, porque Docker Desktop ejecuta los contenedores dentro de una máquina virtual. En OrangePi, arranca con:

```bash
HOST_PROC_SOURCE=/proc HOST_SYS_SOURCE=/sys HOST_ETC_SOURCE=/etc \
NODO_PRIVILEGED=true ENABLE_SYSTEM_ACTIONS=true \
WEB_PORT=80 docker-compose up -d --build
```

### Las acciones del sistema devuelven `501`

Es el comportamiento esperado cuando `ENABLE_SYSTEM_ACTIONS=false`. Solo se habilitan explícitamente en el despliegue Linux de confianza.

## Desarrollo local

Validaciones disponibles:

```bash
node --check frontend/app.js
cd backend && go test ./...
cd .. && docker-compose config
```

## Licencia

MIT. Usa, adapta y despliega NODO libremente.
