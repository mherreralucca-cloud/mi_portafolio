# 🌐 Laboratorio: Despliegue, Hardening y Aislamiento de Servidor DHCP

**Objetivo del Laboratorio:** Implementar, configurar y asegurar un servidor de asignación dinámica de direcciones IP (DHCP) en un entorno Ubuntu Server. La práctica está orientada a automatizar el aprovisionamiento de red para los nodos cliente, garantizando la estabilidad de la infraestructura mediante reservas estáticas. El laboratorio destaca por su enfoque en **Ciberseguridad Defensiva**, aplicando el Principio de Menor Privilegio (PoLP) y confinamiento de procesos mediante la configuración de un entorno aislado (Chroot Jail) para mitigar vectores de ataque.
---
La guia en detalle con el paso a paso y capturas esta en el .pdf adjunto.
---

### Marco Teórico y Arquitectura de Red
El protocolo DHCP (RFC 2131) opera en la Capa de Aplicación del modelo TCP/IP, basando su comunicación en datagramas de la Capa de Transporte mediante los puertos **UDP 67 (Servidor)** y **UDP 68 (Cliente)**. 

Se basa en el **Intercambio DORA**, un flujo de 4 pasos:
1. **D**iscover (Descubrimiento): El cliente envía un broadcast MAC (`FF:FF:FF:FF:FF:FF`) a la red buscando un servidor DHCP.
2. **O**ffer (Oferta): El servidor responde ofreciendo una dirección IP disponible desde su *pool* (rango).
3. **R**equest (Petición): El cliente acepta la IP ofrecida y solicita formalmente su uso.
4. **A**cknowledge (Acuse de recibo): El servidor confirma la asignación, establece el *Lease Time* (tiempo de concesión) y envía la configuración extra (Gateway, DNS).

---

### 1. Instalación y Vinculación de Interfaces
Para proveer el servicio, se despliega la implementación de Internet Systems Consortium (ISC).

```bash
sudo apt update
sudo apt install isc-dhcp-server -y
```

**Asignación de Interfaz Física:**
Un servidor DHCP no debe escuchar a ciegas. Es mandatorio declarar en qué tarjeta de red específica operará el servicio modificando el archivo `/etc/default/isc-dhcp-server`:
```bash
# Declaración de la interfaz de red interna para IPv4
INTERFACESv4="enp0s8"
```

---

### 2. Configuración de Topología y Control de Acceso
El comportamiento del servicio y la estructura lógica de la red se declaran en `/etc/dhcp/dhcpd.conf`.

#### A. Definición del Pool (Subred)
Se establecen los parámetros globales que se inyectarán a los clientes (Gateway, DNS y Máscara):
```text
subnet 192.168.50.0 netmask 255.255.255.0 {
  range 192.168.50.100 192.168.50.200;
  option routers 192.168.50.1;
  option domain-name-servers 8.8.8.8, 1.1.1.1;
  default-lease-time 600;
  max-lease-time 7200;
}
```

#### B. Reservas Estáticas y Hardening Perimetral (MAC Filtering)
Para la infraestructura crítica (switches, impresoras, servidores), se vincula de forma permanente una dirección de Capa 2 (MAC) con una IP de Capa 3. 
Además, se implementa la directiva `deny unknown-clients;` a nivel global. Esto transforma al servidor en un guardián: **si un atacante conecta un equipo no autorizado a un puerto físico del switch, el servidor ignorará su petición DHCP dejándolo sin acceso lógico a la red.**

```text
host servidor_db {
  hardware ethernet 52:54:00:11:22:33;
  fixed-address 192.168.50.10;
}
```

---

### 3. Ciberseguridad: Confinamiento (DHCP Jail)
Si un atacante lograra explotar una vulnerabilidad de *Buffer Overflow* en el servicio DHCP, obtendría acceso al sistema. Para evitarlo, se encapsula el proceso en un árbol de directorios simulado (`chroot`) y se le quitan los privilegios de superusuario (`root`).

```bash
# Creación de la estructura del Jail y ajuste de propietarios al usuario sin privilegios 'dhcpd'
sudo chown -R dhcpd:dhcpd /var/lib/dhcp-jail/var/lib/dhcp
sudo chmod 775 /var/lib/dhcp-jail/var/lib/dhcp
```
*Se ajustan también las reglas de **AppArmor** y los binarios de **systemd** para que el demonio arranque obligatoriamente confinado en esta jaula.*

---

### 4. Gestión del Servicio y Auditoría
Se inicializa el daemon y se audita su estado operativo.

```bash
sudo systemctl restart isc-dhcp-server
```

**Auditoría de Procesos y Puertos:**
1. **Comprobar Aislamiento y Privilegios:**
    ```bash
    ps aux | grep dhcpd
    ```
    *Se verifica que el servicio corra bajo el usuario `dhcpd` (aislado) y no bajo `root`.*
2. **Comprobar Socket de Escucha:**
    ```bash
    sudo ss -ulnp | grep 67
    ```
    *Se verifica que el daemon escuche pasivamente en el puerto `UDP:67` (`0.0.0.0:67`).*

---

### 5. Monitoreo y Resolución de Problemas (Troubleshooting)
* **Base de Datos de Concesiones (`dhcpd.leases`):**
  Es el registro en tiempo real de las asignaciones activas. Se audita mediante `cat /var/lib/dhcp/dhcpd.leases` para inspeccionar qué IP se le entregó a qué *Hardware Address* y en qué marca de tiempo UTC expira.
* **Logs en Vivo (Syslog / Journald):**
  Herramienta vital para diagnosticar negociaciones DORA fallidas o visualizar los bloqueos por `unknown-clients`:
  ```bash
  journalctl -u isc-dhcp-server -f
  ```

---

###  6. Evolución a IPAM
En las redes corporativas a gran escala moderna, el servicio de DHCP no se administra editando archivos de texto manualmente. Se integra dentro de plataformas **IPAM (IP Address Management)** como *NetBox* o *Infoblox*, las cuales combinan DHCP, DNS (BIND) y la documentación de la red bajo una única interfaz gráfica y automatizada mediante APIs.

***

El paso a paso mas en detalle con sus capturas se encuentra en el .pdf adjunto, gracias por ver!

**Bibliografía:**
* Documentación Oficial de Ubuntu Server (ISC DHCP): https://ubuntu.com/server/docs/network-dhcp
* Estándares IETF: RFC 2131 (Protocolo DHCP) y RFC 2132 (Opciones de DHCP).
