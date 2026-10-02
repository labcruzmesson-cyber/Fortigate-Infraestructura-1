# Seguridad de Redes - Implementación y Validación de Túnel IPsec Site-to-Site con FortiGate
# ENLACE HACIA VIDEO: https://youtu.be/rP0XbXDDYTA
## 📌 Datos Generales
- **Autor:** Manuel Alejandro Cruz Messón  
- **Matrícula:** 2025-0689  
- **Fecha:** 2 de Octubre 2026  
- **Plataforma de Simulación:** PNetLab / FortiOS 7.0.3  

---

## 🗺️ Topología de Red
![Topología de Red](https://raw.githubusercontent.com/labcruzmesson-cyber/Fortigate-Infraestructura-1/refs/heads/main/IMAGES/Screenshot%202026-10-02%20132621.png)
## 1. Diseño de Direccionamiento IP y VLSM

El esquema de direccionamiento con máscaras de subred de longitud variable (VLSM) se calculó directamente a partir de la matrícula del autor (**2025-0689**), segmentando los bloques utilizando los octetos **25**, **06** y **89**.

### 1.1 Requisitos de Capacidad por Segmento
1. **Subred 1 (VLAN 10 - Red de Clientes/Usuarios):** Aloja estaciones de trabajo con asignación dinámica vía DHCP. Prefijo `/25`.
2. **Subred 2 (Red LAN Servidor Web - DMZ Remota):** Aloja servidores de producción web (HTTP) bajo directivas de acceso restringido. Prefijo `/28`.
3. **Enlaces de Tránsito WAN (Simulación ISP Pública):** Conexiones Layer 3 entre las interfaces públicas de los firewalls y el entorno upstream.

### 1.2 Tabla de Direccionamiento

| Segmento | ID VLAN | Subred / Prefijo | Máscara de Red | Rango Útil | Puerta de Enlace | Dispositivos Clave Asignados |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **VLAN 10 (Usuarios)** | 10 | `10.25.6.0/25` | `255.255.255.128` | `10.25.6.1 - 10.25.6.126` | `10.25.6.1` | Host Cliente / TinyCore (`10.25.6.10` por DHCP) |
| **LAN Servidor (Producción)** | N/A (Untagged) | `10.25.89.0/28` | `255.255.255.240` | `10.25.89.1 - 10.25.89.14` | `10.25.89.1` | Web Server Ubuntu (`10.25.89.10`), HTTP (Port 80) |
| **WAN FortiGate-A (Usuarios)** | N/A | `192.168.145.0/24` | `255.255.255.0` | IP Pública Asignada | Gateway ISP Upstream | Interfaz `port1` (`192.168.145.147`) |
| **WAN FortiGate-B (Servidor)** | N/A | `192.168.145.0/24` | `255.255.255.0` | IP Pública Asignada | Gateway ISP Upstream | Interfaz `port1` (`192.168.145.148`) |

### 1.3 Configuración de Capa de Enlace (L2 - Switch)
- **Uplink FortiGate (`port2`):** Configurado en modo **Trunk** en el switch de distribución para transportar tramas encapsuladas IEEE 802.1Q (VLAN 10).
- **Puerto de Acceso Cliente:** Configurado en modo **Access** asignado a la VLAN 10 para garantizar el traspaso íntegro de solicitudes DHCP Discover hacia la subinterfaz lógica del firewall.

---

## 2. Configuración en FortiOS (GUI 7.0.3)

### 2.1 Interfaces, DHCP y Enrutamiento Base
- **Subinterfaz VLAN 10 (`FortiGate-A`):**
  - **Ruta GUI:** `Network` > `Interfaces` > `Create New` > `Interface`
  - **Parámetros:** Tipo: VLAN, Interfaz base: `port2`, VLAN ID: `10`, IP/Netmask: `10.25.6.1/255.255.255.128`.
  - **Servidor DHCP:** Habilitado en la subinterfaz, rango asignado: `10.25.6.10 - 10.25.6.120`, DNS: `8.8.8.8`, Lease time: `604800` s.
- **Rutas Estáticas por Defecto (Ambos FortiGates):**
  - Destino: `0.0.0.0/0.0.0.0`
  - Salida: `port1` (WAN)
  - Gateway: Dirección IP del upstream ISP (`192.168.145.2` o provista por el entorno).
- **PAT / NAT Outbound (Salida a Internet):**
  - Política que permite navegación desde `10.25.6.0/25` hacia `port1` con **NAT activado** (`Use Outgoing Interface IP`).

### 2.2 Configuración del Enlace VPN IPsec Site-to-Site
Configurado mediante **IPsec Wizard** con la opción *No NAT between sites*.

| Parámetro | FortiGate-A (Sede Usuarios) | FortiGate-B (Sede Servidor) |
| :--- | :--- | :--- |
| **Remote IP Gateway** | `192.168.145.148` | `192.168.145.147` |
| **Outgoing Interface** | `port1` | `port1` |
| **Authentication** | Pre-shared Key | Pre-shared Key (idéntica) |
| **Phase 2 - Local Subnet** | `10.25.6.0/25` | `10.25.89.0/28` |
| **Phase 2 - Remote Subnet** | `10.25.89.0/28` | `10.25.6.0/25` |

### 2.3 Matriz de Políticas de Firewall

| Nombre de Política | Interfaz Origen | Interfaz Destino | Origen | Destino | Servicio | Acción | NAT |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **VPN_A_SERVER** | `VLAN10_USERS` | `VPN_A_TO_B` (Túnel) | `10.25.6.0/25` | `10.25.89.0/28` | HTTP, PING | ACCEPT | Disabled |
| **VPN_A_USERS** | `VPN_A_TO_B` (Túnel) | `VLAN10_USERS` | `10.25.89.0/28` | `10.25.6.0/25` | ALL | ACCEPT | Disabled |
| **NAT-INTERNET** | `VLAN10_USERS` | `port1` (WAN) | `10.25.6.0/25` | `all` | ALL | ACCEPT | Enabled |
| **Implicit Deny** | `any` | `any` | `all` | `all` | ALL | DENY | N/A |

---

## 3. Verificación y Pruebas Funcionales

### 3.1 Matriz de Comprobaciones

| # | Prueba / Requisito | Acción Ejecutada | Resultado en Terminal | Diagnóstico |
| :-: | :--- | :--- | :--- | :-: |
| **1** | Asignación Dinámica DHCP | Solicitud DHCP en cliente TinyCore | IP asignada: `10.25.6.10/25`, Gateway: `10.25.6.1` | **Éxito** |
| **2** | Enrutamiento y Trazabilidad | `traceroute 10.25.89.10` | 3 saltos transitando la interfaz VPN sin desvíos hacia el ISP público | **Éxito** |
| **3** | Acceso Web Seguro (HTTP - 80) | `curl -I http://10.25.89.10` | `HTTP/1.1 200 OK` (Nginx/1.14.0 Ubuntu) servido a través del canal cifrado | **Éxito** |
| **4** | Traducción NAT a Internet | Salida hacia host externo (`8.8.8.8`) | 0% packet loss; IP origen traducida en logs a `192.168.145.147` | **Éxito** |
| **5** | Aislamiento Estricto (Túnel DOWN) | *Bring Down* manual del túnel en la GUI | Conexión abortada / `Host Unreachable` / `timed out`; sin fuga al ISP | **Éxito** |
| **6** | Recuperación y Restablecimiento | *Bring Up* del túnel en la GUI | Reconexión y entrega inmediata de `HTTP/1.1 200 OK` | **Éxito** |

### 3.2 Salidas de Consola

#### Traza de Red (Traceroute)
```bash
root@box:/home/gns3# traceroute 10.25.89.10
traceroute to 10.25.89.10 (10.25.89.10), 30 hops max, 38 byte packets
 1  10.25.6.1 (10.25.6.1)  1.881 ms  1.508 ms  1.268 ms
 2  192.168.145.148 (192.168.145.148)  1.434 ms  1.712 ms  1.072 ms
 3  10.25.89.10 (10.25.89.10)  1.860 ms  1.866 ms  1.055 ms
```

#### Validación Servicio HTTP (cURL)
```bash
root@box:/home/gns3# curl -I http://10.25.89.10
HTTP/1.1 200 OK
Server: nginx/1.14.0 (Ubuntu)
Date: Fri, 02 Oct 2026 17:39:26 GMT
Content-Type: text/html
Content-Length: 28
Last-Modified: Fri, 02 Oct 2026 16:53:04 GMT
Connection: keep-alive
ETag: "6abfe170-1c"
Accept-Ranges: bytes
```

---

## 4. Análisis Técnico del Comportamiento

1. **Resolución de Saltos (Traceroute):**  
   El salto intermedio responde desde `192.168.145.148` (IP WAN de transporte de FortiGate-B). Cuando el paquete con TTL decreciente es desencapsulado dentro del túnel IPsec, FortiGate-B emite el mensaje `ICMP Time Exceeded` utilizando su interfaz de transporte, certificando que el tráfico fue enrutado por la interfaz virtual IPsec y no por el backbone público general.

2. **Acreditación de Aislamiento y Fuga Cero:**  
   Al derribar el túnel (*Bring Down*), las SA de Fase 2 caducan y FortiOS desmonta dinámicamente la ruta hacia `10.25.89.0/28`. Como no existen políticas que permitan el enrutamiento de prefijos RFC 1918 hacia la interfaz WAN sin NAT, el tráfico hacia el servidor web se descarta de inmediato, impidiendo fugas de paquetes en texto plano hacia el ISP.

---

## 5. Conclusiones

- **Aislamiento Condicional Estricto:** La comunicación con el entorno de producción web depende 100% de la disponibilidad y salud del túnel IPsec.
- **Segmentación VLSM Óptima:** Las subredes dimensionadas a partir de la matrícula `2025-0689` permitieron restringir de manera granular los selectores de tráfico de Fase 2.
- **Hardening y Gestión GUI:** Se comprobó la correcta configuración integral de VLANs 802.1Q, servicios DHCP y directivas de seguridad en FortiOS 7.0.3 sin recurrir a sobreescritura de NAT entre sitios.

---

## 6. Declaración de Uso de IA
En la elaboración de este informe se emplearon herramientas de Inteligencia Artificial Generativa como apoyo de redacción y estructuración. La topología, el diseño de red VLSM, las configuraciones en los equipos de laboratorio y la validación de pruebas fueron implementadas y verificadas por el autor.
