# Laboratorio de Seguridad de Redes – FortiGate + GNS3

**Estudiante:** Gregorys Morel Duluc
**Matrícula:** 2025-0035
**Práctica:** P1 – Seguridad de Redes (ITLA)


## 1. Propósito del laboratorio

Implementar y demostrar una topología segura con un firewall **FortiGate 7.4.12 (VM KVM)** sobre GNS3. La topología segmenta usuarios y servidores, controla el tráfico con políticas, aplica inspección profunda (DPI) e IPS contra inyección SQL, filtra archivos `.exe` y protege contra DoS. El direccionamiento IP se basa en la matrícula **2025-0035** (octeto **35**).

## 2. Topología

![Topología GNS3](diagramas/topologia.png)

| Dispositivo | Rol |
|---|---|
| FortiGate-Final-1 (FortiOS 7.4.12, licencia de evaluación) | Firewall / gateway |
| Switch1 (Ethernet switch GNS3) | VLANs de servidores (1) y usuarios (10) |
| PC1 – WEB-Server | Servidor web (10.0.35.2) |
| PC2 – DB-Server | Servidor de base de datos (10.0.35.3) |
| PC3 – Usuario | Cliente VPCS por DHCP en la VLAN10 |
| Usuario-Linux (Docker Alpine) | Cliente Linux por DHCP en la VLAN10 |
| NAT1 | Salida a Internet (WAN) |
| Cloud1 | Acceso de gestión desde el host (red host-only) |

## 3. Direccionamiento IP

| Red | Subred | Gateway | Interfaz FortiGate |
|---|---|---|---|
| Servidores | 10.0.35.0/28 | 10.0.35.1 | port3 (alias SERVIDORES) |
| Usuarios VLAN10 | 10.0.35.128/25 | 10.0.35.129 | port1 (alias USUARIOS-VLAN10) |
| WAN | DHCP (192.168.42.0/24) | 192.168.42.1 | port2 |
| Gestión | 192.168.56.0/24 | — | IP secundaria de port3: 192.168.56.50 |

| Host | IP |
|---|---|
| WEB-Server | 10.0.35.2/28 |
| DB-Server | 10.0.35.3/28 |
| Usuarios (DHCP) | 10.0.35.130 – 10.0.35.254 |

## 4. Switch – VLANs

| Puerto | Conectado a | Modo | VLAN |
|---|---|---|---|
| 1 | WEB-Server | access | 1 |
| 2 | DB-Server | access | 1 |
| 4 | PC3 (Usuario) | access | 10 |
| 5 | FortiGate port1 | access | 10 |
| 6 | Cloud1 (gestión) | access | 1 |
| 7 | FortiGate port3 | access | 1 |
| 8 | Usuario-Linux | access | 10 |

La seguridad básica del switch consiste en separar usuarios y servidores en VLANs distintas (dominios de broadcast aislados) y usar solo puertos en modo access. El switch integrado de GNS3 no soporta port-security ni otras funciones avanzadas.

![VLANs del switch](capturas/13-switch-vlans.png)

## 5. Configuración del FortiGate (GUI)

### 5.1 Interfaces
![Interfaces](Captura%20de%20pantalla%202026-09-25%20182454.png)

### 5.2 Ruta por defecto y NAT
Ruta estática `0.0.0.0/0` por **port2**, gateway `192.168.42.1`. El NAT se aplica en la política P3 (Usuarios → Internet).
![Rutas](capturas/04-rutas.png)

### 5.3 DHCP en la VLAN10
Servidor DHCP en port1 con el rango 10.0.35.130–254. PC3 recibe la IP 10.0.35.130.
![DHCP PC3](capturas/06-dhcp-pc3.png)

### 5.4 Objetos de dirección
![Direcciones](Captura%20de%20pantalla%202026-09-25%20183653.png)

### 5.5 Políticas de firewall
| # | Nombre | Origen → Destino | Servicio | Acción | Perfiles |
|---|---|---|---|---|---|
| 1 | P1-USR-WEB-HTTPS | USUARIOS → WEB-Server | HTTPS (+HTTP) | ACCEPT | IPS Anti-SQLi, File Filter exe, SSL deep-inspection, log de todas las sesiones |
| 2 | P2-USR-DB-DENY | USUARIOS → DB-Server | MYSQL (3306) | DENY | Log de violaciones |
| 3 | P3-USR-INTERNET | USUARIOS → all (port2) | ALL | ACCEPT + NAT | — |

**WEB-Server solo habla con DB-Server:** no existe ninguna política que permita tráfico con origen en el WEB-Server hacia otra red, así que la *implicit deny* bloquea todo lo que el servidor intente enviar fuera de la red de servidores.

![Políticas](Captura%20de%20pantalla%202026-09-25%20191819.png)

### 5.6 DPI e IPS anti-SQLi con cuarentena
El perfil IPS **Anti-SQLi** incluye las firmas de SQL Injection más una regla por filtro (HTTP, servidor, severidad media/alta/crítica). Las dos reglas tienen acción **Block** y **Quarantine: attacker**. Se aplica junto con el perfil SSL **deep-inspection** (DPI).
![IPS](Captura%20de%20pantalla%202026-09-25%20184443.png)

### 5.7 Filtro de archivos .exe
Perfil File Filter con una regla que bloquea el tipo de archivo `exe`, aplicado en P1.
![File Filter](capturas/09-file-filter-exe.png)

### 5.8 Anti-DoS (rate limiting)
Política DoS en la interfaz de usuarios (port1) con anomalías L3/L4 (tcp_syn_flood, udp_flood, icmp_flood, etc.) en **Block** y logging activo.
![DoS](Captura%20de%20pantalla%202026-09-25%20192102.png)

## 6. Pruebas realizadas

| Prueba | Resultado |
|---|---|
| DHCP en la VLAN10 (PC3 y Usuario-Linux) | ✅ IPs 10.0.35.130 y 10.0.35.131 |
| Usuario → WEB-Server:443 (`ping -P 6 -p 443`) | ✅ Conexión permitida (P1) |
| Usuario → DB-Server:3306 (`ping -P 6 -p 3306`) | ✅ Timeout, bloqueado (P2) |
| Logs de tráfico permitido y denegado | ✅ Ver captura |
| Inyección SQL con cuarentena | ⚠️ Ver nota |

![Pruebas 443/3306](capturas/11-prueba-443-3306.png)
![Logs](capturas/12-log-forward-traffic.png)

**Nota sobre la prueba SQLi:** los payloads se enviaron desde el cliente Usuario-Linux (`wget` con `' OR '1'='1` y `UNION SELECT`). Como el WEB-Server es un VPCS y no implementa un servidor HTTP real, la sesión no llegó a transportar la petición completa, así que no se generó un evento IPS ni una cuarentena que se pudiera mostrar. El perfil Anti-SQLi con cuarentena queda configurado y aplicado a P1, pero su bloqueo en vivo **no se pudo demostrar** en este entorno. Tampoco se ejecutaron pruebas en vivo del filtro `.exe` ni del Anti-DoS. Ambos quedan configurados.

## 7. Notas sobre el entorno

- FortiGate VM 7.4.12 con **licencia de evaluación permanente**. Esa licencia limita la VM a 1 vCPU, 2 GB de RAM, **3 interfaces** y **3 políticas**. Por eso:
  - No fue posible crear una subinterfaz VLAN en el FortiGate (error *"Maximum number of entries has been reached"*). La VLAN10 se implementa en el switch (puertos access) y el FortiGate usa port1 dedicado a esa VLAN.
  - La gestión se movió a una **IP secundaria de port3** (192.168.56.50), para dejar las 3 interfaces a WAN, servidores y usuarios.
- Por falta de memoria en la GNS3 VM, el FortiGate corre con 1400 MB de RAM.
- **Uso de CLI:** la configuración funcional se hizo por GUI. La CLI se usó solo para la puesta en marcha y la resolución de problemas: IP de gestión, DHCP en port2 para activar la licencia, eliminar la interfaz `fortilink` y el NTP asociado, mover la gestión a port3 y agregar la regla por filtro del IPS. Los comandos están en `scripts/fortigate-cli-bootstrap.txt`.

## 8. Capturas del proceso

| Captura | Contenido |
|---|---|
| ![](Captura%20de%20pantalla%202026-09-25%20175940.png) | Dashboard del FortiGate con la licencia activa |
| ![](Captura%20de%20pantalla%202026-09-25%20175259.png) | FortiGate con salida a Internet (ping 8.8.8.8 / google.com) |

Otras capturas del proceso de montaje: [140428](Captura%20de%20pantalla%202026-09-25%20140428.png), [170802](Captura%20de%20pantalla%202026-09-25%20170802.png), [174702](Captura%20de%20pantalla%202026-09-25%20174702.png), [174801](Captura%20de%20pantalla%202026-09-25%20174801.png), [183430](Captura%20de%20pantalla%202026-09-25%20183430.png), [183433](Captura%20de%20pantalla%202026-09-25%20183433.png), [183436](Captura%20de%20pantalla%202026-09-25%20183436.png).

## 9. Archivos del repositorio

- `running-configs/fortigate-running-config.conf`: backup real del FortiGate en funcionamiento
- `scripts/fortigate-cli-bootstrap.txt`: comandos CLI usados
- `scripts/switch-vlan-config.txt`: configuración de VLANs del switch
- `capturas/`: capturas reales del laboratorio
- `diagramas/topologia.png`: topología GNS3

## 10. Referencias
- Fortinet Document Library – FortiOS 7.4 Administration Guide: https://docs.fortinet.com
- Fortinet – FortiGate-VM permanent trial license
