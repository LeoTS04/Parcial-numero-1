# Implementación de Red Corporativa Segura y VPN Site-to-Site

Enlace de demostración en video: https://youtu.be/sHk1NtFDL0c

---

## 1. Propósito

El propósito de este proyecto es diseñar e implementar una arquitectura de red corporativa segura, segmentada mediante VLANs (802.1Q) y protegida a través de un Firewall FortiGate (FortiOS 7.0) como pasarela principal, interconectando una sucursal remota administrada por un router Cisco mediante un túnel VPN IPsec Site-to-Site. Se aplican directivas de endurecimiento de capa 2, enrutamiento estático, traducción de direcciones de red (NAT), control de acceso granular por políticas de firewall e inspección de servicios para mitigar vectores de acceso no autorizados hacia servidores críticos (HTTP y MariaDB).

---

## 2. Diagrama de la Topología

![Topología de Red] <img width="767" height="799" alt="image" src="https://github.com/user-attachments/assets/eefd7d90-c1d6-4770-a3b0-4d87da700872" />


### Esquema de Direccionamiento IP (Matrícula: 2025-0885)

| Dispositivo / Interfaz | Segmento / Red | Dirección IP / Máscara | Descripción |
| :--- | :--- | :--- | :--- |
| **FortiGate port1 (WAN)** | `203.0.113.0/30` | `203.0.113.2/30` | Enlace WAN hacia ISP |
| **FortiGate port2.10** | `10.25.10.0/24` | `10.25.10.1/24` | Gateway VLAN 10 (Usuarios - DHCP) |
| **FortiGate port2.20** | `10.25.20.0/24` | `10.25.20.1/24` | Gateway VLAN 20 (Administrativos) |
| **FortiGate port2.30** | `10.25.85.0/28` | `10.25.85.1/28` | Gateway VLAN Servidores |
| **Web Server (Linux)** | `10.25.85.0/28` | `10.25.85.10/28` | Servidor Web (HTTP:80, SSH:22) |
| **DB Server (Linux)** | `10.25.85.0/28` | `10.25.85.11/28` | Base de Datos (MariaDB:3306, SSH:22) |
| **Switch Cisco IOSvL2** | Gestión / L2 | N/A | Trunk 802.1Q (Nativa 999) |
| **Router ISP (R1) Fa1/0**| `203.0.113.0/30` | `203.0.113.1/30` | Gateway WAN FortiGate |
| **Router ISP (R1) Fa2/0**| `198.51.100.0/30`| `198.51.100.1/30`| Gateway WAN Cisco Sucursal |
| **Cisco Sucursal (R2) Fa0/0**| `198.51.100.0/30`| `198.51.100.2/30`| Enlace WAN hacia ISP |
| **Cisco Sucursal (R2) Fa0/1**| `192.168.85.0/24`| `192.168.85.1/24`| Gateway LAN Sucursal |
| **PC Sucursal** | `192.168.85.0/24`| `192.168.85.10/24`| Host remoto de sucursal |

---

## 3. Descripción de lo Implementado y Evidencias

### 3.1. VLAN por defecto e interfaces sin uso
En el Switch de Capa 2 (Cisco IOSvL2) se modificó la VLAN nativa del enlace troncal hacia la VLAN 999 para mitigar ataques de VLAN Hopping, evitando el uso de la VLAN 1 predeterminada. Asimismo, se deshabilitaron administrativamente todas las interfaces que no tienen un rol asignado en la infraestructura para prevenir conexiones no autorizadas.

* **Evidencia del Trunk y VLAN Nativa 999:**
  ![show interfaces trunk] <img width="630" height="514" alt="image" src="https://github.com/user-attachments/assets/97bfad04-3250-4a7b-aad0-d25f93208816" />


* **Evidencia de Interfaces Apagadas:**
  ![show ip interface brief](img/req1_interfaces_down.png)

---

### 3.2. Salida a Internet y NAT
Se implementó una ruta por defecto en el FortiGate apuntando al ISP (`203.0.113.1`). Se habilitó el servicio DHCP en la subinterfaz de la VLAN 10 para la entrega dinámica de parámetros IP a los clientes. Mediante una política de firewall se habilitó la traducción de direcciones de red (Source NAT con IP de la interfaz saliente), permitiendo que los usuarios de la red interna alcancen destinos externos en Internet con respuesta bidireccional.

* **Conectividad ICMP y Traceroute desde PC de Usuarios:**
  ![Ping a 8.8.8.8](img/req2_ping_pc.png)
  ![Traceroute a 8.8.8.8](img/req2_traceroute_pc.png)

* **Política y Registro de Tráfico con NAT en FortiGate:**
  ![Log de Forward Traffic con NAT](img/req2_fortigate_nat.png)

---

### 3.3. Restricción de Acceso SSH a Servidores
Se establecieron políticas jerárquicas en el FortiGate que restringen la administración de los servidores por el protocolo SSH (TCP/22). Únicamente los dispositivos originados en la VLAN 20 (Administrativos) disponen de una política de aceptación (`ACCEPT`). Todo tráfico SSH proveniente de la VLAN 10 o cualquier otra red es interceptado por una regla de denegación explícita (`DENY`) con registro detallado de eventos.

* **Acceso SSH exitoso desde PC VLAN 20:**
  ![SSH exitoso VLAN 20](img/req3_ssh_vlan20_exito.png)

* **Acceso SSH denegado desde PC VLAN 10:**
  ![SSH denegado VLAN 10](img/req3_ssh_vlan10_bloqueo.png)

* **Registro de logs en FortiGate:**
  ![Logs de Firewall para SSH](img/req3_fortigate_log_ssh.png)

---

### 3.4. Bloqueo de Tráfico de Usuarios hacia Base de Datos (3306)
Se implementó una política de firewall específica para aislar el puerto de servicio MariaDB (TCP/3306) en el DB Server (`10.25.85.11`) respecto a los puestos de trabajo de la VLAN 10. La política aplica acción `DENY` con la directiva de registro de infracciones activa (`Log Violation Traffic`), comprobándose la imposibilidad de apertura de socket TCP desde el PC y la generación del registro correspondiente en el cortafuegos.

* **Intento de conexión bloqueado desde PC VLAN 10:**
  ![Prueba de bloqueo MariaDB] <img width="986" height="98" alt="image" src="https://github.com/user-attachments/assets/c6d46daa-5dcb-41f5-9ed4-bb93c2be2342" />


* **Registro de violación en Logs del FortiGate:**
  ![Log de bloqueo puerto 3306] <img width="708" height="68" alt="image" src="https://github.com/user-attachments/assets/4ddbdde9-53a8-4ad3-9d9d-23a3978578b8" />


---

### 3.5. Servicios Web y Base de Datos
Se configuraron dos entornos Linux dedicados en el segmento `/28`:
1. **Web Server (`10.25.85.10`):** Servicio HTTP activo respondiendo peticiones web por el puerto 80.
2. **DB Server (`10.25.85.11`):** Motor MariaDB configurado con directiva `bind-address = 0.0.0.0`, escuchando peticiones en la interfaz de red sobre el puerto estándar 3306.

* **Estado del servicio Web HTTP:**
  ![Servicio HTTP activo] <img width="1215" height="750" alt="image" src="https://github.com/user-attachments/assets/d10407a2-0cf0-4432-9b14-f12624ef3b87" />


* **Socket de MariaDB a la escucha en puerto 3306:**
  ![MariaDB listening 3306] <img width="1005" height="95" alt="image" src="https://github.com/user-attachments/assets/bf449081-d196-4f44-b0d8-e9d4f7066c3d" />


---

### 3.6. VPN IPsec Site-to-Site entre FortiGate y Cisco
Se estableció un túnel seguro IPsec entre el FortiGate principal y el router Cisco de sucursal. Los parámetros de negociación consisten en IKEv1 (Fase 1: DES, MD5, DH Grupo 5, PSK) y Phase 2 Transform-set (ESP-DES, ESP-MD5). 

Se validó que el tráfico entre el PC de la sucursal (`192.168.85.10`) y el Web Server corporativo (`10.25.85.10`) fluye únicamente por el túnel cifrado. Ante la desconexión administrativa de la VPN, la comunicación cesa por completo, evidenciando que no existen rutas alternas no cifradas.

* **Estado de negociación en Cisco (ISAKMP y IPsec SA):**
  ![show crypto isakmp sa] <img width="492" height="63" alt="image" src="https://github.com/user-attachments/assets/03abe56b-19a0-4acf-beca-01eb1f11c4d7" />

  ![show crypto ipsec sa](img/req7_cisco_ipsec_sa.png)

* **Estado del túnel en la interfaz web de FortiGate:**
  ![Túnel IPsec en Verde](img/req7_fortigate_vpn_up.png)

* **Traza de paquetes (Traceroute) hacia el Servidor Web:**
  ![Traceroute a través de VPN](img/req7_traceroute_vpn.png)

* **Pérdida de conectividad al deshabilitar el túnel:**
  ![Pérdida de paquetes con túnel caído](img/req7_vpn_down_drop.png)

---

## 4. Archivos de Configuración

Los archivos de respaldo con la configuración completa de cada nodo de red se encuentran en el directorio `/configs`:
* `configs/FortiGate.conf`: Configuración exportada desde la interfaz web del FortiGate.
* `configs/Switch_L2.cfg`: Running-config del switch Cisco IOSvL2.
* `configs/Cisco_Sucursal_R2.cfg`: Running-config del router Cisco de sucursal.
* `configs/ISP_R1.cfg`: Running-config del router de tránsito ISP.
