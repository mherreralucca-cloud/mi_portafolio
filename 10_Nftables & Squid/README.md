# 🛡️ Laboratorio: Arquitectura Zero Trust, Firewall (nftables) y Proxy Explícito (Squid)

**Objetivo del Laboratorio:** Diseñar y desplegar una arquitectura de red de borde (Edge) basada en el modelo de Confianza Cero (Zero Trust). La práctica consiste en abandonar el esquema clásico de proxy transparente y adoptar un proxy explícito utilizando Squid, acoplado con nftables para la gestión atómica del firewall. El objetivo principal es restringir por completo la salida directa a Internet de la red interna e inspeccionar a nivel de Capa 7 (SNI) las peticiones HTTPS, logrando un entorno auditado y restrictivo mediante una política de Default DROP.

---

### Marco Teórico: Evolución del Filtrado de Paquetes
A nivel del núcleo de Linux, la gestión de tráfico ha evolucionado para solucionar las deficiencias del pasado:

* **Obsolescencia de iptables:** Por más de 20 años fue el estándar, pero su arquitectura presentaba cuellos de botella:
  1. *Evaluación lineal:* El consumo de CPU se dispara al evaluar listas extensas regla por regla.
  2. *Latencia de red:* Para modificar una regla, se debe copiar toda la tabla, modificarla y reescribirla por completo en el kernel.
  3. *Duplicación de código:* Requería binarios separados y reglas duplicadas para IPv4 (iptables) e IPv6 (ip6tables).
* **El paradigma nftables:** Es el motor de filtrado moderno. Acelera el procesamiento, unifica protocolos y permite operaciones eficientes, donde las reconfiguraciones de reglas son atómicas y dinámicas sin degradar el rendimiento.
* **Proxy Explícito e Inspección SNI:** A diferencia de interceptar tráfico silenciosamente (transparente), el cliente se configura para enrutar su tráfico directo a Squid. Luego, Squid lee el *Server Name Indication* (SNI) de la cabecera TLS para auditar a qué dominio accede el usuario sin necesidad de descifrar el paquete o ejecutar ataques *Man-in-the-Middle*.

---

###  1. Instalación y Dependencias
Para establecer el perímetro, instalamos el motor de filtrado moderno y el proxy.

```bash
sudo apt update
sudo apt install nftables squid curl -y
```

---

### 2. Hardening a Nivel de Red (nftables)
El eje de esta topología es el aislamiento estricto. Se configuran las reglas para que ningún paquete de la red interna cruce hacia Internet a menos que vaya a través del servidor Squid.

* **Activación del Servicio:**
  Se habilita nftables en el arranque para que aplique el *ruleset* al prender el servidor.
  ```bash
  sudo systemctl enable --now nftables
  ```
* **Políticas (Default DROP):** En la tabla de reglas (`/etc/nftables.conf`), se configuran las subredes y se establecen las cadenas de reenvío (FORWARD) en caída por defecto. Cualquier host que intente bypassear el embudo del puerto `3128` (Squid) perderá sus paquetes de forma silenciosa.

---

### 3. Filtrado de Capa 7 y Control de Acceso (Squid)
El tráfico autorizado en Capa 3/4 por nftables es auditado por Squid a nivel de aplicación (Capa 7).

* **Reglas de Acceso (ACL):**
  *Ruta:* `/etc/squid/squid.conf`
  Se definen las Listas de Control de Acceso (ACL) para limitar el uso de la red, bloqueando de forma explícita dominios prohibidos (ej. Redes sociales como Facebook o sitios de entretenimiento).

---

### 4. Auditoría y Verificación de Perímetro
Una vez establecidas las tablas y las listas de control, se recarga el motor de proxy y se inician las pruebas de ruteo.

```bash
sudo systemctl restart squid
```
**Prueba:**
```bash
curl -x http://172.20.0.254:3128 -I https://www.google.com
```
*Resultado esperado:*Tiene que devolver HTTP/1.1 200 OK. El proxy lo dejó pasar a internet

**Prueba de Bloqueo HTTPS:**
Desde una consola cliente ubicada en la red local, se fuerza una petición web apuntando al proxy.

```bash
curl -x http://172.20.0.254:3128 -I https://www.facebook.com
```

*Resultado esperado:* Squid intercepta la solicitud leyendo el SNI de la petición TLS. Al cruzarlo con su ACL restrictiva, interrumpe el acceso y le devuelve al cliente el código `HTTP/1.1 403 Forbidden`. El usuario es incapaz de alcanzar la IP pública.

---

### Conclusión: Arquitectura Zero Trust y Perímetro Corporativo
A lo largo de este laboratorio, superamos la simple instalación de paquetes para diseñar una verdadera arquitectura de borde (Edge). Comprobamos por qué el modelo clásico de proxy transparente y reglas iptables lineales es obsoletto, migrando hacia el motor de nftables y un proxy explícito con inspección SNI para el tráfico HTTPS. 

La red interna ahora carece de acceso directo a Internet, operando bajo un modelo de Confianza Cero donde cada petición de capa 7 es inspeccionada, autorizada o denegada, y registrada para su análisis forense.

---

> **Nota:** El desarrollo paso a paso del laboratorio, el troubleshooting de conectividad, los archivos de configuración exactos y las capturas de pantalla demostrando el funcionamiento de las reglas perimetrales se encuentran documentados en detalle en el archivo `.pdf` adjunto en esta misma carpeta.
