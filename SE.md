##  1. Baliabideen Monitorizazioa eta Kudeaketa

Zerbitzariaren osasuna (RAM, PUZa, Diskoa eta Sarea) bi bideetatik kontrolatu da:

* **Kudeaketa Grafikoa (Cockpit):**
    ```bash
    sudo apt install cockpit -y
    sudo systemctl enable --now cockpit.socket
    # Sarbidea: [https://192.168.70.147:9090](https://192.168.70.147:9090)
    ```
* **Kontsolaren bidezko kudeaketa:**
    Baliabideak xehetasunez aztertzeko tresna hauek erabili dira:
    ```bash
    htop         # RAM eta PUZa monitorizatzeko
    df -h        # Disko gogorraren espazioa ikusteko
    nload        # Sare txartelaren trafikoa aztertzeko
    ```

---

##  2. Kontenedoreak eta Errendimendu Mugak (Docker)

Docker erabili da zerbitzuak isolatzeko, baliabide muga zorrotzak ezarriz:

* **Mugak eta instalazioa:**
    ```bash
    # Kontenedore bat muga zehatzekin abiarazteko adibidea:
    docker run -d --name web-zerbitzaria \
      --cpus=".5" \
      --memory="512m" \
      -p 8080:80 nginx
    ```
* **Monitorizazioa eta Alertak:**
    `docker stats` komandoa erabili da kontenedore bakoitzaren errendimendua web bidez edo kontsolaz aztertzeko eta datuak gordetzeko.

---

##  3. Fitxategi Partekatuak, Baimenak eta Kuotak (Samba)

Samba zerbitzaria konfiguratu da Windows eta Linux bezeroentzat:

* **Konfigurazioa (`/etc/samba/smb.conf`):**
    ```ini
    [Enpresa_Datuak]
    path = /mnt/datos_empresa
    valid users = alex
    writable = yes
    browseable = yes
    ```
* **Baimenak eta Kuotak:**
    ```bash
    # Baimenak zuzendu
    sudo chown -R alex:alex /mnt/datos_empresa
    
    # Kuotak ezarri (500MB muga gogorra)
    sudo edquota -u alex
    # Egiaztapena bezeroetatik (Win/Lin):
    repquota -as /mnt/datos_empresa
    ```

---

##  4. Urrutiko Kudeaketa Grafikoa (RDP)

Zerbitzaria saretik kudeatzeko ingurune grafikoa ezarri da:

* **Zerbitzarian (xrdp + XFCE):**
    ```bash
    sudo apt install xrdp xfce4 xfce4-goodies -y
    # XFCE lehenetsi /etc/xrdp/startwm.sh fitxategian:
    echo "startxfce4" >> /etc/xrdp/startwm.sh
    ```
* **Bezeroak:**
    * **Windows:** *Escritorio Remoto* (MSTSC).
    * **Linux:** **Remmina** aplikazioa, RDP protokoloa erabiliz.

---

##  5. Segurtasuna eta Mantentze Automatizatua (Cron)

Analisia eta eguneratzeak programatu dira:

* **Anti-rootkit (rkhunter):**
    ```bash
    sudo apt install rkhunter -y
    sudo rkhunter --propupd # Datu basea eguneratu
    ```
* **Programazioa (`crontab -e`):**
    ```bash
    # Egunero goizeko 2etan eguneratu eta 3etan rootkit analisia egin
    00 2 * * * apt update && apt upgrade -y
    00 3 * * * rkhunter --check --sk
    ```

---

##  6. Sare Bidezko Instalazio Azpiegitura (PXE)

Bezeroak saretik instalatzeko sistema:

* **DHCP Konfigurazioa (`/etc/dhcp/dhcpd.conf`):**
    ```text
    next-server 192.168.70.147;
    filename "pxelinux.0";
    ```
* **TFTP Zerbitzaria:** `/srv/tftp/` direktorioan boot fitxategiak eta instalazio menuak prestatu dira.

---

##  7. Biltegiratze Azpiegitura (RAID & LVM)

Datuen segurtasuna bermatzeko:

* **RAID 5:** `sudo mdadm --create /dev/md0 --level=5 --raid-devices=3 /dev/sdb /dev/sdc /dev/sdd`
* **LVM:** `pvcreate /dev/md0` -> `vgcreate vg_data /dev/md0` -> `lvcreate -L 10G -n lv_files vg_data`
