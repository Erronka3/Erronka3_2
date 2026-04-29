---
layout: page
title: Sistema Eragileak
---

# 🚀 Proyecto Erronka: Monitorización y Gestión de Servidor Debian

![Debian](https://img.shields.io/badge/Debian-12-A81D33?style=for-the-badge&logo=debian&logoColor=white)
![Docker](https://img.shields.io/badge/docker-%230db7ed.svg?style=for-the-badge&logo=docker&logoColor=white)
![Bash Shell](https://img.shields.io/badge/bash_script-%23121011.svg?style=for-the-badge&logo=gnu-bash&logoColor=white)
![Status](https://img.shields.io/badge/Status-Completado-success?style=for-the-badge)

Este repositorio contiene la documentación completa y los scripts de configuración de un servidor **Debian 12**. Se detallan los procedimientos para la gestión de recursos, compartición de archivos, monitorización avanzada con contenedores, seguridad automatizada y despliegue por red (PXE).

---

## 📋 Tabla de Contenidos

1. [Acceso Remoto Gráfico (XRDP)](#1-acceso-remoto-gráfico-xrdp)
2. [Samba y Cuotas de Disco](#2-samba-y-cuotas-de-disco)
3. [Monitorización (Cockpit, htop, Netdata)](#3-monitorización-cockpit-htop-netdata)
4. [Docker: Límites y Alertas de Recursos](#4-docker-límites-y-alertas-de-recursos)
5. [Seguridad y Automatización (Cron + RKHunter)](#5-seguridad-y-automatización-cron--rkhunter)
6. [Instalación por Red (Servidor PXE)](#6-instalación-por-red-servidor-pxe)

---

## 1. Acceso Remoto Gráfico (XRDP)

Permite la administración visual del servidor desde clientes Windows nativos (RDP) y Linux (Remmina). Se utiliza **XFCE** por ser un entorno de escritorio ligero ideal para servidores.

```bash
# 1. Instalación del entorno XFCE y el servidor XRDP
sudo apt update
sudo apt install xfce4 xfce4-goodies xrdp dbus-x11 -y

# 2. Forzar el uso de XFCE para sesiones remotas
echo "xfce4-session" > ~/.xsession

# 3. Aplicar cambios y habilitar el servicio en el arranque
sudo systemctl restart xrdp
sudo systemctl enable xrdp
```

---

## 2. Samba y Cuotas de Disco

Configuración de una carpeta compartida en la red local con políticas estrictas de almacenamiento por usuario (Quotas) para evitar la saturación del disco raíz.

### 2.1. Despliegue de Samba
```bash
sudo apt install samba smbclient -y
sudo mkdir -p /srv/samba/compartida
sudo chmod 777 /srv/samba/compartida

# Añadir la configuración del recurso compartido al final de smb.conf
sudo bash -c 'cat >> /etc/samba/smb.conf <<EOF

[Compartida]
   path = /srv/samba/compartida
   browseable = yes
   read only = no
   guest ok = yes
EOF'

sudo systemctl restart smbd
```

### 2.2. Implementación de Cuotas (Quotas)
> **Nota:** Requiere modificar `/etc/fstab` añadiendo `,usrquota,grpquota` en las opciones de montaje de la partición raíz `/`.

```bash
sudo apt install quota quotatool -y
sudo mount -o remount /
sudo quotacheck -cum /
sudo quotaon -v /

# Asignar límite de espacio a un usuario específico (ej: jokin)
sudo edquota -u jokin
```

---

## 3. Monitorización (Cockpit, htop, Netdata)

Gestión integral del rendimiento del sistema utilizando herramientas de CLI y Web UI.

* **htop**: Monitorización rápida de procesos por terminal.
* **Cockpit**: Panel de control web general del host (Puerto `9090`).

```bash
sudo apt install htop cockpit -y
sudo systemctl enable --now cockpit.socket
```

---

## 4. Docker: Límites y Alertas de Recursos

Gestión de servicios en contenedores con restricciones de hardware y envío de alertas automáticas ante consumos anómalos.

### Despliegue de Netdata en Contenedor
Se vinculan los volúmenes del sistema anfitrión para permitir una lectura profunda del hardware:
```bash
sudo docker run -d --name=netdata \
  -p 19999:19999 \
  -v /proc:/host/proc:ro \
  -v /sys:/host/sys:ro \
  -v /var/run/docker.sock:/var/run/docker.sock:ro \
  --restart unless-stopped \
  netdata/netdata
```

### Inyección de Alertas Personalizadas (RAM Crítica)
```bash
sudo docker exec -i netdata sh -c "cat >> /etc/netdata/health.d/cgroups.conf" <<EOF
template: cgroup_mem_critico
      on: cgroup.mem_usage
    lookback: 30s
    calc: (\$ram) * 100 / \$mem_usage_limit
   units: %
    warn: \$this > 80
    crit: \$this > 95
    info: ALERTA: Consumo de RAM critico en contenedor. Riesgo de OOM.
EOF

sudo docker restart netdata
```

---

## 5. Seguridad y Automatización (Cron + RKHunter)

Protección proactiva contra intrusiones (Rootkits) y mantenimiento desatendido del sistema.

### Instalación y Actualización de RKHunter
```bash
sudo apt install rkhunter -y
sudo rkhunter --update
sudo rkhunter --propupd
```

### Automatización con Crontab
Programación de escaneos y actualizaciones en horario nocturno:
```bash
sudo bash -c 'cat >> /var/spool/cron/crontabs/root <<EOF
# 02:00 AM - Escaneo de seguridad y generacion de logs
0 2 * * * /usr/bin/rkhunter --check --cronjob >> /var/log/rkhunter_diario.log

# 03:00 AM - Actualizacion completa del sistema (Parches de seguridad)
0 3 * * * apt update && apt upgrade -y
EOF'
```

---

## 6. Instalación por Red (Servidor PXE)

Infraestructura para el despliegue automático de Sistemas Operativos en clientes nuevos mediante la red local (PXE Boot), eliminando la necesidad de medios físicos.

```bash
# 1. Instalación de servicios Core (DHCP/TFTP/Bootloader)
sudo apt install dnsmasq pxelinux syslinux-efi -y
sudo mkdir -p /srv/tftp/pxelinux.cfg

# 2. Preparación de archivos de arranque
sudo cp /usr/lib/PXELINUX/pxelinux.0 /srv/tftp/
sudo cp /usr/lib/syslinux/modules/bios/ldlinux.c32 /srv/tftp/

# 3. Configuración de Dnsmasq
sudo bash -c 'cat >> /etc/dnsmasq.conf <<EOF
interface=enp0s3
dhcp-range=192.168.70.200,192.168.70.250,12h
enable-tftp
tftp-root=/srv/tftp
dhcp-boot=pxelinux.0
EOF'

# 4. Creación del menú de arranque PXE
sudo bash -c 'cat > /srv/tftp/pxelinux.cfg/default <<EOF
DEFAULT menu.c32
PROMPT 0
TIMEOUT 300
MENU TITLE Instalacion por Red (PXE)
LABEL local
  MENU LABEL Arrancar desde disco duro
  LOCALBOOT 0
EOF'

# 5. Aplicar configuración
sudo systemctl restart dnsmasq
sudo systemctl enable dnsmasq
```

---
<div align="center">
  <i>Desarrollado para la resolución de la rúbrica de administración de sistemas.</i>
</div>
