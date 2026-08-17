# 🧱 Laboratorio: Arquitectura Zero Trust, Firewall (nftables) y Proxy Explícito (Squid)

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

### 2. Configuración del Firewall (nftables)
Acá está la base del aislamiento. Modificamos la tabla de reglas para asegurarnos de que la red interna no tenga ruteo directo a Internet. Todo debe morir en el firewall o pasar por el proxy.

**Archivo de configuración:** `/etc/nftables.conf`

Limpiamos las reglas viejas y armamos la estructura base con política DROP:
```text
#!/usr/sbin/nft -f

flush ruleset

table inet filter {
    # Controlamos el tráfico que atraviesa el router (tránsito)
    chain forward {
        type filter hook forward priority 0; policy drop;
    }

    # Controlamos el tráfico que va dirigido al propio servidor
    chain input {
        type filter hook input priority 0; policy drop;
        
        # Permitimos tráfico local
        iif "lo" accept
        
        # Aceptamos conexiones ya establecidas
        ct state established,related accept
        
        # Permitimos que la red interna se conecte al puerto del proxy (3128)
        ip saddr 172.20.0.254/16 tcp dport 3128 accept
        
        # Permitimos administración por SSH
        tcp dport 22 accept
    }
}
```

Habilitamos el servicio para que las reglas persistan al reiniciar:
```bash
sudo systemctl enable --now nftables
```

---

### 3. Filtrado de Capa 7 (Squid)
El tráfico que nftables deja pasar hacia el puerto 3128, ahora lo ataja Squid para auditar a nivel de aplicación.

**Archivo de configuración:** `/etc/squid/squid.conf`

Configuramos Listas de Control de Acceso (ACL) para bloquear dominios específicos (ej: redes sociales) y habilitamos el puerto de escucha:

```text
# Definimos la red local
acl red_interna src 172.20.0.254/16

# Definimos los dominios que queremos bloquear
acl dominios_bloqueados dstdomain .facebook.com .instagram.com

# Aplicamos las reglas de acceso (el orden importa)
http_access deny dominios_bloqueados
http_access allow red_interna
http_access deny all

# Puerto del proxy
http_port 3128
```

Recargamos el servicio para aplicar los cambios:
```bash
sudo systemctl restart squid
```

---

### 4. Pruebas y Verificación del Perímetro
Con el firewall cerrando el paso y el proxy filtrando, simulamos el tráfico de un cliente de la red local intentando entrar a un sitio prohibido.

Ejecutamos `curl` forzando el paso por el proxy hacia una conexión HTTPS:
```bash
curl -x [http://192.168.122.1:3128](http://192.168.122.1:3128) -I [https://www.facebook.com](https://www.facebook.com)
```

*Resultado:* El firewall permite llegar al puerto 3128. Squid lee la petición, identifica el SNI de facebook.com, hace match con la regla `deny` y corta el acceso. La consola nos devuelve el código `HTTP/1.1 403 Forbidden`. El cliente queda totalmente bloqueado de alcanzar la IP pública.

---

### Conclusión: Perímetro Corporativo en la Práctica
En este laboratorio diseñamos una arquitectura de borde (Edge) real. Comprobamos en la consola por qué las viejas reglas de iptables quedaron atrás frente al rendimiento de nftables, y controlamos el tráfico HTTPS usando un proxy explícito con inspección SNI. 

El resultado no es solo un proxy funcionando, sino un entorno de red auditado donde la red interna no tiene salida directa a Internet, operando bajo un modelo de Confianza Cero (Zero Trust).

---

> **Nota:** El desarrollo completo del laboratorio, el troubleshooting, los archivos exactos y las capturas de pantalla de la terminal demostrando los bloqueos se encuentran documentados en detalle en el archivo `.pdf` adjunto en este repositorio.
