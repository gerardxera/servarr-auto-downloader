# Arr Docker Stack (Servarr) — Tutorial de instalación

Este proyecto proporciona una infraestructura completa y automatizada para la gestión, búsqueda y descarga de contenido multimedia, actuando como el *backend* de tu propio servicio de streaming privado. Utilizando el ecosistema "Arr" (Servarr) orquestado a través de Docker Compose, el sistema elimina la necesidad de buscar ficheros manualmente.

El flujo es totalmente autónomo: los usuarios solicitan películas o series a través de una interfaz web amigable (Seerr). Esta petición viaja a los gestores (Radarr/Sonarr), que utilizan un indexador centralizado (Prowlarr) para buscar la mejor versión disponible en la red según tus filtros de calidad e idioma. Una vez encontrada, la orden pasa al cliente de descargas (qBittorrent). Al finalizar, el sistema categoriza, renombra y mueve los ficheros a tu biblioteca definitiva mediante **hardlinks** (atomic moves que ahorran terabytes de espacio en disco), descargando automáticamente los subtítulos exactos (Bazarr). Todo esto ocurre en segundo plano, dejando los archivos listos para que tu servidor multimedia (Jellyfin, Plex o Emby) los reproduzca al instante.

---

### 🛠️ Herramientas incluidas en este stack:

- **qBittorrent** — cliente torrent  
- **Prowlarr** — indexadores centralizados  
- **Radarr** — películas  
- **Sonarr** — series  
- **Lidarr** — música (opcional)  
- **Bazarr** — subtítulos  
- **Seerr** — portal de peticiones (Overseerr/Jellyseerr-compatible)

Diseñado para:

1. **Proxmox** — LXC Docker (recomendado)  
2. **Cualquier host** con Docker + Compose (VM, bare metal, NAS con Docker, etc.)

> [!WARNING]
> **Privacidad / GitHub:** este documento **no** incluye contraseñas, tokens API, claves SSH ni datos de tu lab. Sustituye siempre los placeholders (`YOUR_*`, rutas de ejemplo) por tus valores.

---

## Índice

1. [Qué hace cada herramienta](#1-qué-hace-cada-herramienta)  
2. [Arquitectura y flujo](#2-arquitectura-y-flujo)  
3. [Estructura de carpetas (crítico)](#3-estructura-de-carpetas-crítico)  
4. [Requisitos](#4-requisitos)  
5. [Opción A — Proxmox (LXC Docker)](#5-opción-a--proxmox-lxc-docker)  
6. [Opción B — Docker en cualquier sistema](#6-opción-b--docker-en-cualquier-sistema)  
7. [Compose de referencia](#7-compose-de-referencia)  
8. [Arranque del stack](#8-arranque-del-stack)  
9. [Configuración inicial de cada app](#9-configuración-inicial-de-cada-app)  
10. [Enlazar todo (checklist)](#10-enlazar-todo-checklist)  
11. [DNS, indexadores y salud](#11-dns-indexadores-y-salud)  
12. [Media server externo (Jellyfin / Plex / Emby)](#12-media-server-externo-jellyfin--plex--emby)  
13. [Verificación final](#13-verificación-final)  
14. [Problemas frecuentes](#14-problemas-frecuentes)  
15. [Seguridad y buenas prácticas](#15-seguridad-y-buenas-prácticas)  
16. [Variables del operador](#16-variables-del-operador)

---

## 1. Qué hace cada herramienta

### qBittorrent

Cliente BitTorrent con WebUI.

- Recibe los “grabs” de Radarr/Sonarr/Lidarr.  
- Descarga a carpetas por **categoría** (`movies`, `series`, `anime`, `music`).  
- Las apps *arr* vigilan esas carpetas e **importan** a la biblioteca final.

**No** organiza la biblioteca final: solo descarga.

Puerto típico WebUI: `8080`.

---

### Prowlarr

Hub de **indexadores** (trackers / buscadores de torrents y a veces Usenet).

- Configuras los indexers **una sola vez**.  
- Los sincroniza a Radarr, Sonarr, Lidarr, etc.  
- Si un indexer falla, lo gestionas en Prowlarr y se propaga.

Puerto típico: `9696`.

---

### Radarr

Automatización de **películas**.

- Busca releases según perfiles de calidad.  
- Encarga la descarga a qBittorrent (categoría `movies`).  
- Cuando termina, mueve/copia/hardlink a `Media/Movies` y renombra (estilo Plex/Jellyfin).

Puerto típico: `7878`.

---

### Sonarr

Igual que Radarr, pero para **series de TV** (y a menudo anime en root folder aparte).

- Categoría qBit: `series` (y opcionalmente `anime`).  
- Biblioteca: `Media/Series`, `Media/Anime`.

Puerto típico: `8989`.

---

### Bazarr

Descarga **subtítulos** automáticamente.

- Se conecta a Radarr y Sonarr.  
- Lee los mismos ficheros de `Media/` y busca subs (OpenSubtitles, etc., según configures).  
- Idiomas habituales en ES: `es`, `es-MX` / latino, etc.

Puerto típico: `6767`.

---

### Seerr (Overseerr / Jellyseerr family)

Portal de **peticiones** amigable.

- Usuarios piden películas/series desde el móvil o el navegador.  
- Seerr habla con Radarr/Sonarr (y con el media server: Jellyfin/Plex/Emby).  
- No descarga por sí mismo: delega en el stack *arr*.

Puerto típico: `5055`.

> **Nota:** Seerr necesita un **media server** externo para el wizard completo. No es obligatorio instalar Jellyfin en el mismo host que el stack *arr*.

---

### Lidarr (opcional, recomendado si quieres música automática)

Como Radarr/Sonarr, pero para **música** (artistas/álbumes).

- Categoría qBit: `music`.  
- Biblioteca: `Media/Music`.  
- Un reproductor (p. ej. SwingMusic, Navidrome, Jellyfin) solo **lee** esa carpeta; no descarga.

Puerto típico: `8686`.

---

### Qué no va en este stack (por diseño)

| App | Motivo |
|-----|--------|
| Jellyfin / Plex / Emby | Mejor en otro LXC/VM; este CT se centra en descarga + organización |
| Portainer / Arcane | Opcionales para UI de Docker; el stack se opera con `docker compose` |

---

## 2. Arquitectura y flujo

```text
Usuario / Seerr
      │
      ▼
Prowlarr  ──sync indexers──►  Radarr / Sonarr / Lidarr
                                  │
                                  │  “Grab” (categoría movies|series|anime|music)
                                  ▼
                            qBittorrent
                                  │
                                  │  descarga en Torrents/<categoría>
                                  ▼
                         Radarr/Sonarr/Lidarr
                                  │
                                  │  import + rename
                                  ▼
                         Media/Movies|Series|Anime|Music
                                  │
                                  ▼
                    Jellyfin / SwingMusic / Plex (otro host)
                                  │
                                  ▼
                               Bazarr (subs sobre Media/)
```

**Regla de oro de rutas:** dentro de los contenedores todo vive bajo `/data/...`.  
Nunca configures en la UI de Radarr/Sonarr rutas del host del tipo `/mnt/...`.

---

## 3. Estructura de carpetas (crítico)

El fallo #1 de estos stacks es: **paths inconsistentes** o carpetas que no existen en el bind mount.

### 💡 La magia de esta estructura: Hardlinks y Atomic Moves

El mayor beneficio de montar una única ruta raíz (`/data`) que contenga tanto los torrents como la biblioteca final, es que permitimos al sistema operativo realizar **hardlinks** (enlaces duros). 

Cuando qBittorrent finaliza una descarga en `/data/torrents/` y Radarr/Sonarr la importa a `/data/movies/`, **el archivo no se duplica**. Ambas rutas apuntarán al mismo bloque de datos físicos en tu disco. Esto hace que la importación sea instantánea y te ahorra terabytes de almacenamiento. Si usaras volúmenes separados (ej. un volumen para descargas y otro para multimedia), Docker obligaría a copiar el archivo completo cada vez, duplicando el espacio ocupado.

### 3.1 Árbol canónico (host)

Sustituye `LIBRARY_ROOT` por tu ruta real.

Ejemplo de lab (solo ejemplo):

```text
LIBRARY_ROOT=/mnt/storage/Homelab/Biblioteca
```

Árbol:

```text
LIBRARY_ROOT/
├── Media/
│   ├── Movies/            # biblioteca final películas (Radarr)
│   ├── Series/            # series (Sonarr)
│   ├── Anime/             # anime (Sonarr root opcional)
│   ├── Documentaries/     # opcional
│   ├── Music/             # música (Lidarr)
│   │   ├── _Incoming/     # opcional, staging manual
│   │   └── Various Artists/
│   └── Peliculas/         # legado; preferir Movies
└── Torrents/
    ├── movies/            # qBit categoría movies
    ├── series/
    ├── anime/
    ├── music/
    ├── peliculas/         # legado
    ├── temp/              # descargas incompletas
    │   ├── movies/
    │   ├── series/
    │   └── music/
    └── complete/          # opcional
        ├── movies/
        ├── series/
        ├── anime/
        └── music/
```

### 3.2 Crear el árbol

```bash
# Ajusta LIBRARY_ROOT
export LIBRARY_ROOT="/mnt/storage/Homelab/Biblioteca"

mkdir -p "$LIBRARY_ROOT"/{Media/{Movies,Series,Anime,Documentaries,Music/_Incoming,Music/"Various Artists",Peliculas},Torrents/{movies,series,anime,music,peliculas,temp/{movies,series,music},complete/{movies,series,anime,music}}}

# Permisos orientativos (ajusta UID/GID a tu PUID/PGID del compose)
# En muchos mounts de NAS verás nobody:nogroup; lo importante es escritura real.
chmod -R a+rwX "$LIBRARY_ROOT" 2>/dev/null || true
```

### 3.3 Mapa host → contenedor

| Uso | Ruta en el host | Ruta dentro del contenedor |
|-----|-----------------|----------------------------|
| Config de cada app | `./radarr`, `./sonarr`, … | `/config` |
| Películas finales | `.../Media/Movies` | `/data/movies` |
| Series finales | `.../Media/Series` | `/data/series` |
| Anime final | `.../Media/Anime` | `/data/anime` |
| Música final | `.../Media/Music` | `/data/music` |
| Todo Media (Bazarr) | `.../Media` | `/data/media` |
| Torrents (raíz qBit) | `.../Torrents` | `/data/torrents` |
| Incompletos | `.../Torrents/temp` | `/data/torrents/temp` |
| Cat. movies | `.../Torrents/movies` | `/data/torrents/movies` |
| Cat. series | `.../Torrents/series` | `/data/torrents/series` |
| Cat. anime | `.../Torrents/anime` | `/data/torrents/anime` |
| Cat. music | `.../Torrents/music` | `/data/torrents/music` |

### 3.4 Categorías qBittorrent

Fichero típico (ruta depende de la imagen; en hotio suele estar bajo el volumen de config):

```json
{
  "movies":  { "save_path": "/data/torrents/movies" },
  "series":  { "save_path": "/data/torrents/series" },
  "anime":   { "save_path": "/data/torrents/anime" },
  "music":   { "save_path": "/data/torrents/music" }
}
```

En qBit WebUI:

- Default save path: `/data/torrents`  
- Temp/incomplete: `/data/torrents/temp`  

**Nunca** pongas rutas del host (`/mnt/...`) dentro de qBit o de las *arr*.

---

## 4. Requisitos

### Hardware orientativo

| Recurso | Mínimo razonable | Cómodo |
|---------|------------------|--------|
| CPU | 2 vCPU | 4 vCPU |
| RAM | 2–4 GB | 4–8 GB |
| Disco root (OS + configs Docker) | 16–32 GB | 32 GB |
| Biblioteca / torrents | Disco grande o NAS bind | HDD/SSD externo o share |

La biblioteca **no** debe vivir solo en el rootfs pequeño del LXC: usa bind mount o volumen de datos.

### Software

- Docker Engine + Docker Compose v2+  
- DNS público usable (muchos DNS de ISP bloquean dominios de indexers)  
- Red LAN estable; firewall permitiendo los puertos que expongas  

### Imágenes de referencia

Este tutorial usa imágenes **hotio** (`ghcr.io/hotio/...`), habituales en homelab.  
También valen **linuxserver** u oficiales; cambia nombres de env vars y paths de config según su documentación.

Variables comunes hotio/linuxserver:

```yaml
PUID: 1000          # UID del usuario que escribe en el disco
PGID: 1000
TZ: Europe/Madrid   # tu zona horaria
UMASK: "002"
```

Averigua PUID/PGID:

```bash
id -u
id -g
```

---

## 5. Recomendado: Despliegue en Proxmox (LXC Docker)

Si planeas desplegar este stack dentro de un contenedor LXC en Proxmox (la opción recomendada para Homelabs), hemos preparado una guía dedicada para mantener este documento principal más limpio. 

En esa guía explicamos cómo utilizar los **scripts de la comunidad de Proxmox** (Proxmox VE Helper-Scripts) para crear fácilmente un contenedor *unprivileged* con Docker nativo, además de cómo configurar los DNS y asignar el almacenamiento (bind mounts) correctamente.

👉 **[Leer la guía detallada de instalación en Proxmox LXC](PROXMOX_LXC_GUIDE.md)**

---

## 6. Instalación estándar con Docker en cualquier sistema

Válido para: Debian/Ubuntu bare metal, VM (Proxmox VM, cloud), TrueNAS Scale (apps/Docker), Unraid, etc.

### 6.1 Instalar Docker

Sigue la documentación oficial de tu distro:

- https://docs.docker.com/engine/install/

Comprueba:

```bash
docker run --rm hello-world
docker compose version
```

### 6.2 Usuario y permisos

```bash
sudo usermod -aG docker "$USER"
# cierra sesión y vuelve a entrar
```

Define PUID/PGID del usuario que poseerá los ficheros de media.

### 6.3 Biblioteca

- Disco local: crea `LIBRARY_ROOT` en ese disco.  
- NFS/SMB: monta el share y usa esa ruta en los binds del compose.  
- Asegura que el UID del contenedor puede **escribir**.

### 6.4 Layout de proyecto

```text
~/arr-stack/   (o /opt/arr-stack)
├── compose.yaml
├── .env                 # NO subir a Git (secretos)
├── .env.example         # sí subir
├── radarr/
├── sonarr/
├── prowlarr/
├── qbittorrent/
├── bazarr/
├── seerr/
└── lidarr/
```

Añade a `.gitignore`:

```gitignore
.env
**/config.xml
**/*credentials*
**/qBittorrent.conf
backups/
```

---

## 7. Compose de referencia

Crea `compose.yaml` (ajusta **todas** las rutas `LIBRARY_ROOT` y puertos si colisionan).

```yaml
x-common-env: &common-env
  PUID: 1000
  PGID: 1000
  TZ: Europe/Madrid
  UMASK: "002"

services:

  radarr:
    image: ghcr.io/hotio/radarr:latest
    container_name: radarr
    restart: unless-stopped
    ports:
      - "7878:7878"
    environment: *common-env
    volumes:
      - ./radarr:/config
      - ${LIBRARY_ROOT}/Media/Movies:/data/movies
      - ${LIBRARY_ROOT}/Media/Documentaries:/data/documentaries
      - ${LIBRARY_ROOT}/Torrents/movies:/data/torrents/movies
    networks:
      - media

  sonarr:
    image: ghcr.io/hotio/sonarr:latest
    container_name: sonarr
    restart: unless-stopped
    ports:
      - "8989:8989"
    environment: *common-env
    volumes:
      - ./sonarr:/config
      - ${LIBRARY_ROOT}/Media/Series:/data/series
      - ${LIBRARY_ROOT}/Media/Anime:/data/anime
      - ${LIBRARY_ROOT}/Torrents/series:/data/torrents/series
      - ${LIBRARY_ROOT}/Torrents/anime:/data/torrents/anime
    networks:
      - media

  lidarr:
    image: ghcr.io/hotio/lidarr:latest
    container_name: lidarr
    restart: unless-stopped
    ports:
      - "8686:8686"
    environment: *common-env
    volumes:
      - ./lidarr:/config
      - ${LIBRARY_ROOT}/Media/Music:/data/music
      - ${LIBRARY_ROOT}/Torrents/music:/data/torrents/music
    networks:
      - media

  bazarr:
    image: ghcr.io/hotio/bazarr:latest
    container_name: bazarr
    restart: unless-stopped
    ports:
      - "6767:6767"
    environment: *common-env
    volumes:
      - ./bazarr:/config
      - ${LIBRARY_ROOT}/Media:/data/media
    networks:
      - media

  prowlarr:
    image: ghcr.io/hotio/prowlarr:latest
    container_name: prowlarr
    restart: unless-stopped
    ports:
      - "9696:9696"
    environment: *common-env
    volumes:
      - ./prowlarr:/config
    networks:
      - media

  qbittorrent:
    image: ghcr.io/hotio/qbittorrent:latest
    container_name: qbittorrent
    restart: unless-stopped
    ports:
      - "8080:8080"
      # Opcional: puerto BitTorrent (TCP/UDP). Ábrelo en el router solo si quieres mejor conectividad.
      # - "6881:6881"
      # - "6881:6881/udp"
    environment:
      PUID: 1000
      PGID: 1000
      UMASK: "002"
      TZ: Europe/Madrid
      WEBUI_PORTS: 8080/tcp
      # NO pongas la contraseña real en el repo. Usa .env:
      # WEBUI_PASSWORD: ${QBIT_WEBUI_PASSWORD}
      LIBTORRENT: v1
    volumes:
      - ./qbittorrent:/config
      - ${LIBRARY_ROOT}/Torrents:/data/torrents
      - ${LIBRARY_ROOT}/Torrents/temp:/data/torrents/temp
    networks:
      - media

  seerr:
    image: ghcr.io/hotio/seerr:latest
    container_name: seerr
    restart: unless-stopped
    ports:
      - "5055:5055"
    environment: *common-env
    volumes:
      - ./seerr:/config
    networks:
      - media

networks:
  media:
    name: media_network
```

### `.env` (local, no commitear)

```bash
# .env.example → copiar a .env y rellenar
LIBRARY_ROOT=/mnt/storage/Homelab/Biblioteca
PUID=1000
PGID=1000
TZ=Europe/Madrid
QBIT_WEBUI_PASSWORD=change-me-to-a-strong-password
```

Si usas `WEBUI_PASSWORD` en el compose, referencia `${QBIT_WEBUI_PASSWORD}`.

> [!WARNING]
> **Nota sobre el cambio de contraseña en qBittorrent:** 
> La imagen de *hotio* aplica la variable `WEBUI_PASSWORD` la primera vez que se inicia el contenedor. Sin embargo, **no sobrescribirá** la contraseña si ya existe un archivo de configuración previo en el volumen de configuración. Si modificas esta clave en el archivo `.env` cuando el stack ya ha sido arrancado alguna vez, la nueva contraseña será ignorada.
> 
> **Solución:** Si pierdes el acceso o cambias la clave a posteriori, deberás borrar la configuración antigua de qBittorrent con el contenedor parado, o cambiarla directamente desde la interfaz web (WebUI) y luego actualizar las credenciales en los *Download Clients* de Radarr, Sonarr y Lidarr.

---

## 8. Arranque del stack

```bash
cd /root/docker   # o tu ruta de proyecto
# export LIBRARY_ROOT=...  o usa fichero .env

# Crear dirs de config
mkdir -p radarr sonarr prowlarr qbittorrent bazarr seerr lidarr backups

> **¿Por qué creamos estas carpetas a mano antes de arrancar?** 
> Si ejecutas `docker compose up` y Docker no encuentra los directorios locales definidos en los volúmenes (`./radarr`, `./sonarr`, etc.), los creará automáticamente usando el usuario `root`. Dado que las imágenes de *hotio* están configuradas para usar tu usuario estándar (`PUID=1000` / `PGID=1000`), los contenedores fallarán al arrancar por falta de permisos de escritura. Al crear las carpetas tú mismo previamente con `mkdir`, estas heredan tus permisos de usuario correctos.

# Crear árbol de biblioteca (sección 3)
# ...

docker compose pull
docker compose up -d
docker compose ps
```

Prueba escritura:

```bash
docker exec radarr touch /data/movies/.write_test && docker exec radarr rm /data/movies/.write_test
docker exec qbittorrent touch /data/torrents/temp/.write_test && docker exec qbittorrent rm /data/torrents/temp/.write_test
```

URLs (sustituye `ARR_HOST` por la IP/hostname del host):

| Servicio | URL |
|----------|-----|
| qBittorrent | `http://ARR_HOST:8080` |
| Prowlarr | `http://ARR_HOST:9696` |
| Radarr | `http://ARR_HOST:7878` |
| Sonarr | `http://ARR_HOST:8989` |
| Lidarr | `http://ARR_HOST:8686` |
| Bazarr | `http://ARR_HOST:6767` |
| Seerr | `http://ARR_HOST:5055` |

---

## 9. Configuración inicial de cada app

Hazlo en este **orden**.

### 9.1 qBittorrent

1. Entra al WebUI; cambia la contraseña por defecto si aplica.  
2. Options → Downloads:
   - Default save path: `/data/torrents`  
   - Keep incomplete in: `/data/torrents/temp`  
3. Categories: `movies`, `series`, `anime`, `music` con save paths de la sección 3.4.  
4. Anota usuario/contraseña WebUI para las *arr*.  
5. (Opcional) Connection: puertos BT; no expongas el WebUI a Internet sin reverse proxy + auth fuerte.

### 9.2 Prowlarr

1. Wizard: auth **Forms**, usuario/contraseña fuertes.  
2. Settings → Indexers: añade indexers **públicos que pasen el Test**.  
3. No insistas en sitios detrás de Cloudflare agresivo o trackers muertos.  
4. Settings → Apps: se configurará **después** de tener API keys de Radarr/Sonarr/Lidarr.  
5. Copia la **API Key** (Settings → General).

### 9.3 Radarr

1. Auth Forms + usuario/contraseña.  
2. Media Management → Root Folders: **`/data/movies`** (Add).  
3. Download Clients → qBittorrent:
   - Host: `qbittorrent` (nombre del **servicio Docker**, no `localhost` si están en la misma red compose)  
   - Port: `8080`  
   - Username / Password: los del WebUI  
   - Category: `movies`  
   - Test → debe ser OK  
4. Settings → General → copia **API Key**.  
5. Quality profiles: elige el perfil que uses (p. ej. HD-1080p).  
6. (Opcional) Preferencia de idioma: ver sección 11.3.

### 9.4 Sonarr

Igual que Radarr:

- Root folders: `/data/series` y opcional `/data/anime`  
- Download client: host `qbittorrent`, category `series` (y otro client o misma app con category `anime` si separas)  
- API Key para Prowlarr  

### 9.5 Lidarr (si lo usas)

- Root folder: `/data/music`  
- Download client qBit, category **`music`**  
- API de Lidarr es **v1** (no v3)  
- API Key → Prowlarr  

### 9.6 Prowlarr → Apps (sync)

En Prowlarr → Settings → Apps → Add:

| App | Sync URL | Notas |
|-----|----------|--------|
| Radarr | `http://radarr:7878` | Full sync; API key de Radarr |
| Sonarr | `http://sonarr:8989` | Full sync |
| Lidarr | `http://lidarr:8686` | Full sync; categorías audio |

Test cada app. Luego Sync / ApplicationIndexerSync.

### 9.7 Bazarr

1. Auth + idiomas de subtítulos deseados.  
2. Sonarr/Radarr:
   - Hostnames Docker: `sonarr` / `radarr`  
   - API keys  
   - Ruta media en Bazarr: ya tiene `/data/media` montado → mapea a lo que Bazarr espere según su UI (a menudo el path que ven Sonarr/Radarr).  
3. Providers de subtítulos: configura cuentas donde hagan falta (no pongas credenciales en el README del repo).

### 9.8 Seerr

1. Completa el wizard cuando tengas:
   - Media server (Jellyfin/Plex/Emby) alcanzable  
   - Radarr + Sonarr con API keys  
2. Si aún no hay media server: puedes dejar Seerr a medias; el stack de descarga ya funciona sin él.

---

## 10. Enlazar todo (checklist)

```text
[ ] Carpetas Media/* y Torrents/* existen y son escribibles
[ ] compose up -d → todos Up
[ ] qBit paths solo /data/torrents...
[ ] Radarr root /data/movies + DC test OK (category movies)
[ ] Sonarr root /data/series + DC test OK (category series)
[ ] Lidarr root /data/music + DC test OK (category music)   [si aplica]
[ ] Prowlarr apps fullSync + indexers visibles en cada *arr
[ ] Bazarr conecta a Radarr/Sonarr
[ ] Prueba: añadir 1 película en Radarr → aparece en qBit cat. movies → importa a Media/Movies
[ ] DNS del host/CT no es el filtro del ISP
[ ] Ningún secreto en git status
```

### Prueba funcional mínima (película)

1. Radarr → Add Movie → elige un título de prueba.  
2. Monitored + perfil de calidad.  
3. Search / Interactive Search → Grab de un release con seeders.  
4. qBit: torrent en categoría `movies`.  
5. Al completar: Radarr Activity → importado; fichero en `Media/Movies/...`.  
6. Health de Radarr sin errores de Download Client / Remote Path Mapping.

---

## 11. DNS, indexadores y salud

### 11.1 DNS

Si Prowlarr no resuelve nombres:

1. Cambia nameservers del host/CT a `1.1.1.1`, `8.8.8.8`, `9.9.9.9`.  
2. Reinicia.  
3. `getent hosts` de un dominio de indexer.

### 11.2 Indexadores

| Situación | Acción |
|-----------|--------|
| Test OK en Prowlarr | Mantener y sync |
| Cloudflare / timeout eterno | No insistir en bucle; quitar del health |
| Tracker privado | Credenciales del usuario en Prowlarr UI (nunca en git) |
| Indexer solo películas | No esperes que alimente Sonarr |

### 11.3 Preferencia de español (orientativo)

- En Radarr, poner el quality profile en idioma **Spanish** como filtro duro puede dejar muchas películas en *Missing* si no hay release ES.  
- Enfoque más robusto: idioma del perfil **Any** + **Custom Formats** que sumen score a tags `spanish`, `latino`, `castellano`, etc., con `minFormatScore = 0`.  
- Asegura al menos un indexer con contenido en español si priorizas ES.

### 11.4 Health endpoints (debug)

Desde el host Docker (red interna o localhost):

```bash
# Sustituye API keys — no las guardes en scripts del repo
curl -sS -H "X-Api-Key: $RADARR_KEY" http://127.0.0.1:7878/api/v3/health
curl -sS -H "X-Api-Key: $SONARR_KEY" http://127.0.0.1:8989/api/v3/health
curl -sS -H "X-Api-Key: $LIDARR_KEY" http://127.0.0.1:8686/api/v1/health
curl -sS -H "X-Api-Key: $PROWLARR_KEY" http://127.0.0.1:9696/api/v1/health
```

Errores típicos:

- **RemotePathMapping**: alguna app ve rutas `/mnt/...` que no existen en el contenedor → unifica a `/data/...` y borra mappings obsoletos.  
- **DownloadClientUnavailable**: mal host/password/categoría.  
- **IndexerStatusCheck**: indexer caído o DNS.

---

## 12. Media server externo (Jellyfin / Plex / Emby)

Recomendación de arquitectura:

```text
CT/VM A  →  *arr + qBit + Prowlarr + Bazarr + Seerr
CT/VM B  →  Jellyfin (monta el mismo Media/ en solo lectura o RW)
CT/VM C  →  reproductor de música (opcional) montando Media/Music
```

Ambos hosts deben ver el **mismo** `LIBRARY_ROOT` (mismo NFS, mismo disco bind, etc.).

En Jellyfin:

- Library Movies → carpeta host equivalente a `Media/Movies`  
- Library Shows → `Media/Series`  
- Escaneo tras imports de Radarr/Sonarr  

En un player de música:

- Root **dentro del contenedor del player** (p. ej. `/music`), no la ruta larga del NAS si no está montada así.

Seerr → Settings → Media Server + Radarr/Sonarr.

---

## 13. Verificación final

```bash
docker compose ps
docker compose logs --tail=50 radarr sonarr prowlarr qbittorrent
```

| Check | OK si… |
|-------|--------|
| Contenedores | Estado `Up` |
| Puertos LAN | HTTP 200/302 en 7878, 8989, 9696, 8080, 6767, 5055, 8686 |
| Root folders | “Accessible” en cada *arr |
| qBit categories | movies/series/anime/music → `/data/torrents/...` |
| Import test | Fichero real en `Media/...` con tamaño coherente |
| Git | `git status` sin `.env`, sin `config.xml`, sin passwords |

---

## 14. Problemas frecuentes

### Contenedor `Exited (128)` / mount permission denied

Docker intenta crear el path del bind y falla.

- La ruta del **host** en el compose no existe (typo: falta un directorio intermedio).  
- Crea el árbol completo **antes** de `compose up`.  
- Recreate: `docker compose up -d --force-recreate`

### Descarga OK pero no importa

1. Categoría qBit ≠ la configurada en Radarr/Sonarr.  
2. Remote path mapping incorrecto.  
3. Permisos: el PUID del *arr* no puede leer el torrent completado.  
4. qBit aún “moving”; espera o revisa tamaños en Torrents vs Media.

### *arr no autentica contra qBit

- Password distinto al del download client.  
- Variable `WEBUI_PASSWORD` de la imagen no aplicó sobre conf ya existente → parar contenedor, ajustar conf o password vía API, arrancar, actualizar clients.

### Prowlarr sin indexers útiles

- DNS del ISP.  
- Cloudflare.  
- Tracker offline.  
Limpia health rojo; deja solo indexers que pasen Test.

### Seerr redirige a `/setup`

Wizard incompleto o sin media server. No bloquea Radarr/Sonarr.

---

## 15. Seguridad y buenas prácticas

1. **No subas a GitHub:** `.env`, API keys, `config.xml`, passwords qBit, dumps de auth.  
2. WebUI solo en **LAN** o detrás de reverse proxy (SSO/Authelia, TLS).  
3. No publiques `8080`/`7878`/… a Internet en crudo.  
4. Contraseñas distintas por servicio o un gestor de secretos.  
5. Backup periódico de `/config` de cada app + listado de indexers.  
6. Respeta copyright y leyes de tu jurisdicción: este stack es infraestructura técnica; el contenido es responsabilidad del operador.  
7. Actualiza imágenes con control: `docker compose pull && docker compose up -d` tras backup.

---

## 16. Variables del operador

Rellena al adaptar este repo a tu entorno (no hace falta publicar los valores reales):

| Variable | Descripción | Ejemplo de forma |
|----------|-------------|------------------|
| `ARR_HOST` | IP o DNS del host Docker | `192.168.x.y` |
| `VMID` | ID LXC en Proxmox | entero libre |
| `LIBRARY_ROOT` | Raíz Media+Torrents | `/mnt/.../Biblioteca` |
| `PUID` / `PGID` | Dueño de ficheros | `1000`/`1000` |
| `TZ` | Zona horaria | `Europe/Madrid` |
| Bridge / storage PVE | Red y disco del CT | `vmbr0`, `local-lvm`, … |
| Media server URL | Jellyfin/Plex | otro host:puerto |

---

## Apéndice A — Comandos útiles

```bash
# Estado
docker compose ps
docker compose logs -f sonarr

# Parar / arrancar
docker compose stop
docker compose up -d

# Recrear un servicio
docker compose up -d --force-recreate radarr

# Backup configs
tar -czf backups/arr-$(date +%F).tgz compose.yaml .env radarr sonarr prowlarr qbittorrent bazarr seerr lidarr

# Dentro de un contenedor
docker exec -it radarr bash
ls -la /data/movies /data/torrents/movies
```

## Apéndice B — Proxmox: checklist rápido

```text
[ ] LXC Docker unprivileged + nesting
[ ] IP fija + DNS públicos
[ ] mp bind del storage de biblioteca
[ ] Árbol Media/ y Torrents/ creado
[ ] /root/docker/compose.yaml + .env
[ ] docker compose up -d
[ ] Apps enlazadas por nombre DNS Docker
[ ] vzdump o backup de configs
[ ] Documentación interna con secretos fuera de git
```

## Apéndice C — Mapa de puertos

| Puerto | Servicio |
|-------:|----------|
| 8080 | qBittorrent WebUI |
| 9696 | Prowlarr |
| 7878 | Radarr |
| 8989 | Sonarr |
| 8686 | Lidarr |
| 6767 | Bazarr |
| 5055 | Seerr |

---

## Licencia del documento

Puedes copiar y adaptar este README en tu repositorio.  
Las aplicaciones citadas tienen sus propias licencias (Radarr/Sonarr/Prowlarr/Lidarr/Bazarr/qBittorrent/Seerr/hotio images). Revisa cada proyecto upstream antes de redistribuir imágenes o binarios.

---

**Resumen en una frase:** unifica rutas bajo `LIBRARY_ROOT` → `Media/` + `Torrents/`, monta todo como `/data/...` en Docker, enlaza Prowlarr → *arr → qBittorrent por nombres de servicio, y mantén secretos fuera de GitHub.
