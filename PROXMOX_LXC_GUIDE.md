# Guía de Despliegue en Proxmox VE (LXC)

Este documento es una extensión del [tutorial principal (README.md)](README.md) y detalla los pasos específicos para preparar el entorno ideal en Proxmox utilizando un contenedor LXC *unprivileged* y los scripts de la comunidad. 

Una vez finalices esta preparación, deberás volver al README principal para arrancar el stack de Servarr.

---

## 1. Idea y Arquitectura Base

Un contenedor LXC **unprivileged** configurado con:

- `nesting=1` (para permitir la ejecución de Docker dentro del LXC).
- IP estática en tu *bridge* (normalmente `vmbr0`).
- *Bind mount* del almacenamiento donde reside tu biblioteca (`mp0`).
- Motor de Docker y Docker Compose para levantar el stack.

> **Nota:** No instales tu servidor multimedia (Jellyfin/Plex) en este mismo contenedor salvo que sea estrictamente necesario. Es mejor mantener los roles separados.

## 2. Crear el LXC Docker (Proxmox VE Helper-Scripts)

En el **nodo Proxmox** (desde la shell del host PVE), la forma más limpia y estandarizada es utilizar el script oficial de Docker LXC de [community-scripts](https://community-scripts.github.io/ProxmoxVE/):

```bash
# Ejecutar en el nodo PVE, no dentro de otro guest a ciegas
bash -c "$(curl -fsSL https://raw.githubusercontent.com/community-scripts/ProxmoxVE/main/ct/docker.sh)"
```

Durante el asistente elige (orientativo):

| Parámetro | Sugerencia |
|-----------|------------|
| VMID | Libre (ej. el que tú decidas) |
| Hostname | `arrdocker` o similar |
| Unprivileged | Sí |
| Nesting / keyctl | Sí (necesarios para Docker) |
| CPU / RAM / disco | 2–4 vCPU, 4 GB RAM, 32 GB rootfs |
| Red | IP estática + gateway + **DNS 1.1.1.1 / 8.8.8.8** (importante) |
| Storage rootfs | El almacenamiento rápido que uses para contenedores |

Anota estos datos:

- El `VMID` asignado.
- La IP del contenedor (nos referiremos a ella como `ARR_HOST` ).
- La contraseña root del contenedor (guárdala en tu gestor de contraseñas, **nunca** en el repositorio).

### 3. Montar la biblioteca (bind)

En el host PVE, asumimos que tu disco de almacenamiento o NAS ya está montado, por ejemplo, en `/mnt/storage`.

Ejecuta lo siguiente para pasar ese almacenamiento al interior del contenedor:

```bash
# Ejemplo — AJUSTA paths y VMID
pct set YOUR_VMID -mp0 /mnt/storage,mp=/mnt/storage,backup=0
pct reboot YOUR_VMID
```

Dentro del contenedor LXC debe existir ahora la misma ruta de montaje, ej. `/mnt/storage/...`.

Debes crear el árbol de carpetas explicado en la sección 3 del README dentro del contenedor (o directamente en el host sobre ese montaje, ya que es el mismo disco físico).

>Aviso sobre API: En Proxmox, los bind mounts (`mp0`) generalmente solo los puede fijar el usuario `root@pam` utilizando `pct set`. Si administras Proxmox mediante una API con un token de privilegios limitados (`privsep=1`), asegúrate de realizar este paso manualmente como root en el nodo.

### 4. DNS del LXC (imprescindible para indexers)

Muchos servidores DNS de operadores de internet bloquean dominios de trackers de torrents. Si no cambias esto, Prowlarr dará errores de "Name does not resolve".

En la configuración de red del contenedor LXC (desde la interfaz web de Proxmox o modificando el archivo del contenedor), asegura que tienes DNS públicos:

```text
nameserver: 1.1.1.1 8.8.8.8 9.9.9.9
```

Reinicia el contenedor y verifica la resolución:

```bash
cat /etc/resolv.conf
getent hosts one.one.one.one
```

### 5 Verificación de Docker dentro del CT

Si utilizaste el script de la comunidad en el Paso 2, Docker ya debería estar instalado y funcionando. Entra a la consola del contenedor y compruébalo:

```bash
docker --version
docker compose version
systemctl is-active docker
```

(Si por algún motivo no estuviera instalado, sigue la documentación oficial de Docker para Debian/Ubuntu).

```bash
# Debian/Ubuntu — sigue la doc oficial de Docker
# https://docs.docker.com/engine/install/
```

### 6 Directorio de trabajo del stack

Prepara la carpeta donde residirá la configuración del stack:

```bash
mkdir -p /root/docker
cd /root/docker
# En este directorio es donde colocaremos el compose.yaml y las carpetas de configuración.
```
✅ El entorno LXC ya está preparado.

👉 Regresa a la [sección 7](#7-compose-de-referencia) del README principal para crear el archivo compose.yaml y levantar los servicios.

### 7 Snapshot / backup antes de cambios grandes

Antes de realizar cambios grandes en el futuro, ten en cuenta:

- Si usas almacenamiento tipo `dir` con discos `.raw`: el snapshot nativo de Proxmox(PVE) **no** es fiable. Utiliza copias de seguridad estándar **vzdump** del contenedor completo cuando el stack esté estable.
- Adicionalmente, puedes hacer copias de seguridad regulares de las configuraciones de Docker (tarball):

```bash
mkdir -p /root/docker/backups
tar -czf /root/docker/backups/arr-config-$(date +%F).tgz \
  -C /root/docker compose.yaml radarr sonarr prowlarr qbittorrent bazarr seerr lidarr 2>/dev/null
```
