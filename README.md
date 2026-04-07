# PXE Deploy RPi4

Servidor de arranque por red (PXE) basado en Raspberry Pi 4 con Docker.
Permite instalar sistemas operativos de forma desatendida en equipos de la red local, tanto con BIOS clásica como con UEFI.

## ¿Qué hace este proyecto?

Cuando un equipo arranca por red, la RPi le asigna una IP, le sirve un menú de arranque y lanza la instalación o herramienta elegida — todo sin tocar el equipo físicamente.

**Sistemas operativos disponibles:**
- Debian 13 CLI (instalación desatendida)
- Debian 13 GUI con GNOME (instalación desatendida)
- Windows 11 Pro (WinPE + autounattend.xml)

**Herramientas de diagnóstico:**
- SystemRescue
- Clonezilla
- Memtest86+

**Monitorización (opcional):**
- Grafana — dashboards de métricas y logs
- Prometheus — métricas del sistema y contenedores
- Loki + Promtail — logs centralizados
- Portainer — gestión visual de contenedores

---

## Arquitectura

```
Equipo cliente
     │
     │  1. "Necesito una IP"  (broadcast DHCP)
     ▼
┌─────────────┐
│   dnsmasq   │  ← Reparte IPs y sirve ficheros de arranque (TFTP)
└─────────────┘
     │
     │  2. Descarga kernel + menú
     ▼
┌─────────────────────────────────────────────┐
│                  MENÚ PXE                   │
│  Debian CLI │ Debian GUI │ Windows │ Tools  │
└─────────────────────────────────────────────┘
     │
     ├── Debian  →  nginx (HTTP) → preseed automático
     └── Windows →  samba (SMB) → ISO expandida + autounattend.xml
```

**Stack de monitorización:**
```
Node Exporter ──┐
                ├──► Prometheus ──► Grafana (métricas)
cAdvisor ───────┘

Promtail ──────────► Loki ───────► Grafana (logs)

Portainer: gestión visual de contenedores
```

---

## Requisitos previos

- Raspberry Pi 4 (2 GB RAM o más recomendado)
- Raspbian OS Lite (64-bit)
- Docker instalado
- La RPi conectada por cable Ethernet a la red donde están los equipos cliente
- Un switch o hub entre la RPi y los equipos (no conectar directamente a un router con DHCP activo, o desactivar el DHCP del router)

---

## Instalación

### 1. Clonar el repositorio

```bash
git clone https://github.com/tu-usuario/pxe-deploy-rpi4.git
cd pxe-deploy-rpi4
```

### 2. Configurar las variables de entorno

```bash
cp .env.example .env
nano .env
```

Edita los valores según tu entorno:

| Variable | Descripción | Valor por defecto |
|---|---|---|
| `RPI_IP` | IP fija de la RPi | `192.168.100.1` |
| `DHCP_RANGE_START` | Primera IP del rango DHCP | `192.168.100.50` |
| `DHCP_RANGE_END` | Última IP del rango DHCP | `192.168.100.200` |
| `NETWORK_IFACE` | Interfaz de red de la RPi | `eth0` |
| `GRAFANA_ADMIN_USER` | Usuario de Grafana | `admin` |
| `GRAFANA_ADMIN_PASSWORD` | Contraseña de Grafana | — |
| `SAMBA_USER` | Usuario del share Windows | `pxeuser` |
| `SAMBA_PASSWORD` | Contraseña del share Windows | — |

> **Importante:** El fichero `.env` contiene contraseñas. Nunca lo subas a GitHub. Está en el `.gitignore`.

### 3. Preparar los ficheros de arranque

Estos ficheros son binarios que hay que obtener manualmente. El script de instalación crea la estructura de carpetas pero no los descarga.

**PXELINUX (para BIOS clásica):**
```bash
sudo apt install pxelinux syslinux-common
cp /usr/lib/PXELINUX/pxelinux.0 tftpboot/
cp /usr/lib/syslinux/modules/bios/menu.c32 tftpboot/
cp /usr/lib/syslinux/modules/bios/ldlinux.c32 tftpboot/
```

**Debian 13 (netboot):**
```bash
# Descarga el tarball de netboot de Debian 13
wget https://deb.debian.org/debian/dists/trixie/main/installer-amd64/current/images/netboot/netboot.tar.gz
tar -xf netboot.tar.gz
cp debian-installer/amd64/linux tftpboot/debian-cli/vmlinuz
cp debian-installer/amd64/initrd.gz tftpboot/debian-cli/initrd.gz
# Para la versión GUI, usa los mismos ficheros (el preseed controla qué se instala)
cp tftpboot/debian-cli/vmlinuz tftpboot/debian-gui/vmlinuz
cp tftpboot/debian-cli/initrd.gz tftpboot/debian-gui/initrd.gz
```

**Windows 11 (WinPE + ISO):**
- Genera el WinPE con el [ADK de Windows](https://learn.microsoft.com/es-es/windows-hardware/get-started/adk-install)
- Copia los ficheros del WinPE a `tftpboot/winpe/`
- Expande la ISO de Windows 11 en `samba/win11/`
- Coloca el `autounattend.xml` dentro de `samba/win11/`

**SystemRescue:**
```bash
# Descarga la ISO desde https://www.system-rescue.org/
# y extrae el kernel e initrd
```

**Clonezilla:**
```bash
# Descarga desde https://clonezilla.org/downloads.php
# y extrae el kernel e initrd
```

**Memtest86+:**
```bash
sudo apt install memtest86+
cp /boot/memtest86+.bin tftpboot/memtest86/memtest
```

### 4. Ejecutar el script de instalación

```bash
sudo bash scripts/install.sh
```

El script:
- Comprueba que Docker está instalado
- Configura la IP estática en la RPi (`/etc/dhcpcd.conf`)
- Crea los directorios necesarios con los permisos correctos
- Copia los menús PXE al directorio TFTP
- Levanta todos los contenedores

> Después de ejecutarlo, **reinicia la RPi** para que se aplique la IP estática.

---

## Uso

### Levantar los contenedores

**Con monitorización (recomendado):**
```bash
docker compose -f docker-compose.yml -f docker-compose.monitor.yml up -d
```

**Solo el stack PXE:**
```bash
docker compose -f docker-compose.yml up -d
```

### Parar los contenedores

```bash
docker compose -f docker-compose.yml -f docker-compose.monitor.yml down
```

### Ver los logs

```bash
# Logs de dnsmasq (para ver qué equipos piden IP)
docker logs -f dnsmasq

# Logs de samba
docker logs -f samba

# Logs de nginx
docker logs -f nginx
```

---

## Acceso a los servicios

Una vez levantado, los servicios están disponibles en:

| Servicio | URL | Descripción |
|---|---|---|
| Grafana | `http://192.168.100.1:3000` | Dashboards de métricas y logs |
| Portainer | `http://192.168.100.1:9000` | Gestión visual de contenedores |
| Prometheus | `http://192.168.100.1:9090` | UI de métricas (avanzado) |
| nginx (preseeds) | `http://192.168.100.1/preseed/` | Ficheros preseed de Debian |

> Cambia `192.168.100.1` por el valor de `RPI_IP` en tu `.env` si lo modificaste.

### Añadir dashboards a Grafana

1. Busca un dashboard en [grafana.com/grafana/dashboards](https://grafana.com/grafana/dashboards)
2. Descarga el JSON
3. Colócalo en `config/grafana/provisioning/dashboards/`
4. Reinicia Grafana: `docker restart grafana`

El dashboard aparecerá automáticamente sin configuración adicional.

---

## Estructura del proyecto

```
pxe-deploy-rpi4/
│
├── .env.example                     # Plantilla de variables (copia a .env)
├── .gitignore
├── docker-compose.yml               # Stack PXE principal
├── docker-compose.monitor.yml       # Stack de monitorización
│
├── config/
│   ├── dnsmasq/
│   │   └── dnsmasq.conf             # DHCP + TFTP
│   ├── prometheus/
│   │   └── prometheus.yml           # Scrape targets
│   ├── loki/
│   │   └── loki.yml                 # Almacenamiento de logs
│   ├── promtail/
│   │   └── promtail.yml             # Recogida de logs
│   └── grafana/
│       └── provisioning/
│           ├── datasources/
│           │   └── datasources.yml  # Prometheus + Loki como fuentes
│           └── dashboards/
│               └── dashboards.yml   # Carga automática de dashboards
│
├── pxe/
│   ├── menus/
│   │   ├── pxelinux.cfg/
│   │   │   └── default              # Menú BIOS clásica
│   │   └── grub/
│   │       └── grub.cfg             # Menú UEFI
│   └── unattended/
│       ├── preseed-cli.cfg          # Instalación desatendida Debian CLI
│       └── preseed-gui.cfg          # Instalación desatendida Debian GUI
│
├── scripts/
│   └── install.sh                   # Script de instalación inicial
│
├── tftpboot/                        # Ficheros de arranque (no en git)
│   ├── pxelinux.cfg/
│   ├── grub/
│   ├── debian-cli/
│   ├── debian-gui/
│   ├── winpe/
│   ├── systemrescue/
│   ├── clonezilla/
│   └── memtest86/
│
└── samba/
    └── win11/                       # ISO Windows expandida (no en git)
```

---

## Solución de problemas

**El equipo no recibe IP por DHCP**
- Comprueba que `dnsmasq` está corriendo: `docker ps`
- Revisa los logs: `docker logs dnsmasq`
- Asegúrate de que no hay otro servidor DHCP en la red (router, por ejemplo)
- Verifica que la interfaz en `dnsmasq.conf` coincide con la real: `ip a`

**El equipo recibe IP pero no arranca el menú PXE**
- Comprueba que los ficheros `pxelinux.0` y `menu.c32` están en `tftpboot/`
- Para UEFI: comprueba que `grubx64.efi` está en `tftpboot/`
- Revisa que el equipo tiene el arranque por red activado en la BIOS

**La instalación de Debian falla**
- Verifica que el preseed es accesible: `curl http://192.168.100.1/preseed/preseed-cli.cfg`
- Comprueba que `nginx` está corriendo: `docker ps`

**Windows no puede acceder al share SMB**
- Verifica que `samba` está corriendo: `docker logs samba`
- Desde el WinPE, prueba: `net use \\192.168.100.1\win11 /user:pxeuser tu_contraseña`
- Asegúrate de que la ISO de Windows está expandida en `samba/win11/`

---

## Licencia

MIT