---
layout: page
title: Sistema Eragileak
---
#  Erronka Proiektua: Debian Zerbitzariaren Monitorizazioa eta Kudeaketa

![Debian](https://img.shields.io/badge/Debian-12-A81D33?style=for-the-badge&logo=debian&logoColor=white)
![Docker](https://img.shields.io/badge/docker-%230db7ed.svg?style=for-the-badge&logo=docker&logoColor=white)
![Bash Shell](https://img.shields.io/badge/bash_script-%23121011.svg?style=for-the-badge&logo=gnu-bash&logoColor=white)
![Status](https://img.shields.io/badge/Egoera-Osatuta-success?style=for-the-badge)

Repositorio honek **Debian 12** zerbitzari baten konfigurazio-scriptak eta dokumentazio osoa barne hartzen ditu. Bertan azaltzen dira erabiltzaileen kudeaketa, diskoen RAID konfigurazioa, baliabideen monitorizazio aurreratua, segurtasun automatizatua eta sare bidezko inplementazioa (PXE).

---

##  Edukien Aurkibidea

1. [Erabiltzaileen Kudeaketa](#1-erabiltzaileen-kudeaketa)
2. [RAID-aren Konfigurazioa eta Muntaketa](#2-raid-aren-konfigurazioa-eta-muntaketa)
3. [Urruneko Sarbide Grafikoa (XRDP)](#3-urruneko-sarbide-grafikoa-xrdp)
4. [Samba eta Disko-kuotak](#4-samba-eta-disko-kuotak)
5. [Monitorizazioa (Cockpit, htop, Netdata)](#5-monitorizazioa-cockpit-htop-netdata)
6. [Docker: Baliabideen Mugak eta Alertak](#6-docker-baliabideen-mugak-eta-alertak)
7. [Segurtasuna eta Automatizazioa (Cron + RKHunter)](#7-segurtasuna-eta-automatizazioa-cron--rkhunter)
8. [Sare bidezko Instalazioa (PXE Zerbitzaria)](#8-sare-bidezko-instalazioa-pxe-zerbitzaria)

---

## 1. Erabiltzaileen Kudeaketa

Sistemarako sarbidea kontrolatzeko, erabiltzaile lokalak sortu dira eta baimen egokiak esleitu zaizkie.

```bash
# Talde berria sortu (adibidez, 'garatzaileak')
sudo groupadd garatzaileak

# Erabiltzaile berria sortu (adibidez, 'jokin')
sudo adduser jokin

# Erabiltzailea sortutako taldera gehitu
sudo usermod -aG garatzaileak jokin

# Erabiltzaileari administratzaile baimenak eman (sudo taldera gehitu)
sudo usermod -aG sudo jokin

# Taldeak eta erabiltzaileak ondo sortu direla egiaztatu
groups jokin
```

---

## 2. RAID-aren Konfigurazioa eta Muntaketa

Datuen segurtasuna eta erabilgarritasuna bermatzeko, RAID 1 (Ispilua / Mirror) konfiguratu da bi disko gehigarrirekin (`sdb` eta `sdc`), eta sistema abiaraztean automatikoki munta dadin ezarri da.

### 2.1. RAID-a sortzea (`mdadm`)
```bash
# mdadm tresna instalatu
sudo apt update
sudo apt install mdadm -y

# RAID 1 array-a sortu bi diskorekin
sudo mdadm --create --verbose /dev/md0 --level=1 --raid-devices=2 /dev/sdb /dev/sdc

# Fitxategi-sistema (ext4) eman RAID-ari
sudo mkfs.ext4 /dev/md0
```

### 2.2. Karpetak Muntatzea (`/etc/fstab`)
```bash
# Datuak gordetzeko karpeta sortu
sudo mkdir -p /mnt/raid_datuak

# RAID-a eskuz muntatu lehenengo aldiz
sudo mount /dev/md0 /mnt/raid_datuak

# Sistema berrabiaraztean automatikoki munta dadin /etc/fstab-en gehitu
sudo bash -c 'echo "/dev/md0 /mnt/raid_datuak ext4 defaults 0 0" >> /etc/fstab'

# Konfigurazioa gorde RAID-a hurrengo abioetan ezagutzeko
sudo mdadm --detail --scan | sudo tee -a /etc/mdadm/mdadm.conf
sudo update-initramfs -u
```

---

## 3. Urruneko Sarbide Grafikoa (XRDP)

Zerbitzariaren administrazio bisuala ahalbidetzen du Windows (RDP) eta Linux (Remmina) bezeroetatik. **XFCE** erabiltzen da ingurune grafiko arin bezala.

```bash
# XFCE ingurunearen eta XRDP zerbitzariaren instalazioa
sudo apt install xfce4 xfce4-goodies xrdp dbus-x11 -y

# Urruneko saioetan XFCE erabiltzera behartu
echo "xfce4-session" > ~/.xsession

# Aldaketak aplikatu eta zerbitzua abiaraztean gaitu
sudo systemctl restart xrdp
sudo systemctl enable xrdp
```

---

## 4. Samba eta Disko-kuotak

Sare lokalean partekatutako karpeta baten konfigurazioa, biltegiratze-politika zorrotzekin erabiltzaileko (Kuotak).

### 4.1. Sambaren Inplementazioa
```bash
sudo apt install samba smbclient -y
sudo mkdir -p /srv/samba/partekatua
sudo chmod 777 /srv/samba/partekatua

# Gehitu partekatutako baliabidearen konfigurazioa smb.conf-en amaieran
sudo bash -c 'cat >> /etc/samba/smb.conf <<EOF

[Partekatua]
   path = /srv/samba/partekatua
   browseable = yes
   read only = no
   guest ok = yes
EOF'

sudo systemctl restart smbd
```

### 4.2. Kuoten Inplementazioa (Quotas)
> **Oharra:** Ezinbestekoa da `/etc/fstab` fitxategia editatzea, `/` erro-partizioaren muntatze-aukeretan `,usrquota,grpquota` gehituz.

```bash
sudo apt install quota quotatool -y
sudo mount -o remount /
sudo quotacheck -cum /
sudo quotaon -v /

# Esleitu espazio-muga erabiltzaile zehatz bati (adib: jokin)
sudo edquota -u jokin
```

---

## 5. Monitorizazioa (Cockpit, htop, Netdata)

Sistemaren errendimenduaren kudeaketa integrala CLI eta Web UI tresnak erabiliz.

* **htop**: Terminal bidezko prozesuen monitorizazio azkarra.
* **Cockpit**: Ostalariaren web-kontrol panela (`9090` portua).
* **Netdata**: Monitorizazio metriko sakona denbora errealean. Hardwarearen eta Docker kontenedoreen egoera aztertzen du, alerta sistemarekin batera (`19999` portua).

```bash
sudo apt install htop cockpit -y
sudo systemctl enable --now cockpit.socket
```

---

## 6. Docker: Baliabideen Mugak eta Alertak

Kontenedoreetan dauden zerbitzuen kudeaketa, hardware-murrizketekin eta kontsumo anomaloen aurrean alerta automatikoak bidaliz.

### Netdata Kontenedorean Hedatzea
```bash
sudo docker run -d --name=netdata \
  -p 19999:19999 \
  -v /proc:/host/proc:ro \
  -v /sys:/host/sys:ro \
  -v /var/run/docker.sock:/var/run/docker.sock:ro \
  --restart unless-stopped \
  netdata/netdata
```

### Alerta Pertsonalizatuak Txertatzea (RAM Kritikoa)
```bash
sudo docker exec -i netdata sh -c "cat >> /etc/netdata/health.d/cgroups.conf" <<EOF
template: cgroup_mem_critico
      on: cgroup.mem_usage
    lookback: 30s
    calc: (\$ram) * 100 / \$mem_usage_limit
   units: %
    warn: \$this > 80
    crit: \$this > 95
    info: ALERTA: RAM kontsumo kritikoa kontenedorean. OOM arriskua.
EOF

sudo docker restart netdata
```

---

## 7. Segurtasuna eta Automatizazioa (Cron + RKHunter)

Intrusioen aurkako babes proaktiboa (Rootkit-ak) eta sistemaren mantentze-lan automatizatua.

### RKHunter-en Instalazioa
```bash
sudo apt install rkhunter -y
sudo rkhunter --update
sudo rkhunter --propupd
```

### Automatizazioa Crontab bidez
```bash
sudo bash -c 'cat >> /var/spool/cron/crontabs/root <<EOF
# 02:00 AM - Segurtasun-eskaneatzea eta log-ak sortzea
0 2 * * * /usr/bin/rkhunter --check --cronjob >> /var/log/rkhunter_diario.log

# 03:00 AM - Sistemaren eguneratze osoa (Segurtasun-partxeak)
0 3 * * * apt update && apt upgrade -y
EOF'
```

---

## 8. Sare bidezko Instalazioa (PXE Zerbitzaria)

Sistema Eragileak sare lokalaren bidez (PXE Boot) bezero berrietan automatikoki hedatzeko azpiegitura.

```bash
# Oinarrizko zerbitzuen instalazioa
sudo apt install dnsmasq pxelinux syslinux-efi -y
sudo mkdir -p /srv/tftp/pxelinux.cfg

# Abio-fitxategiak prestatzea
sudo cp /usr/lib/PXELINUX/pxelinux.0 /srv/tftp/
sudo cp /usr/lib/syslinux/modules/bios/ldlinux.c32 /srv/tftp/

# Dnsmasq-en konfigurazioa
sudo bash -c 'cat >> /etc/dnsmasq.conf <<EOF
interface=enp0s3
dhcp-range=192.168.70.200,192.168.70.250,12h
enable-tftp
tftp-root=/srv/tftp
dhcp-boot=pxelinux.0
EOF'

# PXE abio-menuaren sortzea
sudo bash -c 'cat > /srv/tftp/pxelinux.cfg/default <<EOF
DEFAULT menu.c32
PROMPT 0
TIMEOUT 300
MENU TITLE Sare bidezko Instalazioa (PXE)
LABEL local
  MENU LABEL Disko gogorretik abiarazi
  LOCALBOOT 0
EOF'

# Konfigurazioa aplikatu
sudo systemctl restart dnsmasq
sudo systemctl enable dnsmasq
```

---
<div align="center">
  <i>Sistemen administrazio erronkaren errubrika ebazteko garatua.</i>
</div>
