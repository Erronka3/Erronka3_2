---

layout: page

title: Sareen planifikazioa

---



## 1. Sarearen Diseinua

Gure sarearen topologia desberdinetan banatuta dago konektibitatea eta segurtasuna bermatzeko.

![SAREAREN DISEINUA](img/Sareen%20planifikazioa/DISEINUA3.2.png)

## 2. Packet tracer

packet tracerrean egin dugu gure sarea, ondoren fisikoan errezago egiteko



![Packet tracer](img/Sareen%20planifikazioa/packettracer.png)



## 3. Sareen Taula (VLAN)

Hona hemen sare bakoitzaren xehetasunak:



| Red / VLAN | Descripción | Dirección de Red | CIDR | Máscara |

| :--- | :--- | :--- | :--- | :--- |

| **GELAKO SAREA** | Red General / Server0 | 192.168.70.0 | /24 | 255.255.255.0 |

| **SISTEMAK DHCP** | Red de Server2 | 192.168.50.0 | /24 | 255.255.255.0 |

| **OTS AREA** | Red de Access Point0 | 192.168.40.0 | /27 | 255.255.255.224 |

| **SISTEMAK DMZ** | Mongo/NodeRed | 192.168.30.0 | /27 | 255.255.255.224 |

| **SAMBA SAREA** | Red de Server3 | 192.168.60.0 | /28 | 255.255.255.240 |

| **LANGILEAK** | Red IPv6 | 2001:DB8:ACAD:20:: | /64 | N/A (IPv6) |



## 4. Konfigurazioa eta Bideratzea

**NAT MikroTik-en:** Source NAT (Masquerade) konfiguratuta dago ikasgelako sarerantz internetera irteera emateko

* [cite_start]**Bideratze Purua:** Samba sarean NAT gabe funtzionatzen du barne-ikusgarritasuna errazteko



## 5. IPv6 Zerbitzua (DHCPv6)

Langile sarean DHCPv6 zerbitzaria gehitu da helbideak automatikoki banatzeko

**Pool izena:** POOL-LANGILEAK 

**DNS:** 001:4860:4860::8888 

