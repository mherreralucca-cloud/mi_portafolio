# 🌐 Laboratorio: Arquitectura, Despliegue y Hardening de Servidor DNS (BIND9)

**Objetivo del Laboratorio:** Implementar un servidor de nombres de dominio (DNS) maestro utilizando BIND9 en un entorno Ubuntu Server. La práctica no solo abarca la resolución directa e inversa para una red corporativa (`aula1.lan`), sino que pone un énfasis crítico en el aseguramiento del servicio. Se aplican políticas de ciberseguridad avanzada mediante la inhabilitación de recursión pública y el confinamiento del proceso (Chroot Jail) gestionado por AppArmor, previniendo escaladas de privilegios en caso de explotación.

---

### Marco Teórico y Topología Estructural
El protocolo DNS es la base de datos distribuida más grande del mundo, estructurada en un árbol jerárquico estricto:
* **Zona Raíz (Root Zone):** Representada lógicamente por un punto (`.`). Existen 13 servidores raíz a nivel mundial replicados mediante enrutamiento Anycast.
* **TLD (Top-Level Domains):** Dominios de nivel superior genéricos (`.com`, `.org`) o territoriales (`.ar`, `.es`).
* **SLD (Second-Level Domains):** El dominio comercial o corporativo (`aula1.lan`).
* **Zonas de Resolución:**
  * *Directa:* Traduce nombres legibles por humanos (ej. `www.aula1.lan`) a direcciones IP.
  * *Inversa:* Traduce una IP a su nombre de host correspondiente, utilizando el dominio especial `in-addr.arpa`. Es vital para auditorías y validación de correos (Antispam).

---

### 1. Instalación y Dependencias
Para aprovisionar la infraestructura de resolución, utilizamos BIND9, el estándar de facto en servidores UNIX/Linux, junto con las herramientas de diagnóstico de red.

```bash
sudo apt update
sudo apt install bind9 dnsutils net-tools -y
```

---

### 2. Hardening: Confinamiento del Demonio (AppArmor)
Por defecto, BIND corre con demasiados privilegios en el sistema. Para mitigar vectores de ataque, se modifica la política de control de acceso mandatorio (MAC) del kernel mediante AppArmor, restringiendo exactamente qué directorios y descriptores de archivos puede tocar el proceso `named`.

*En este laboratorio, nos aseguramos de que el servicio BIND9 no pueda salir de su directorio de trabajo asignado, previniendo que un ataque comprometa la raíz del sistema operativo.*

---

### 3. Configuración Global y de Zonas
Toda la lógica de BIND9 reside en el directorio `/etc/bind/`. El despliegue se divide en tres niveles de configuración:

#### A. Opciones Globales (`named.conf.options`)
Se definen los *Forwarders* (a quién preguntarle si no sabemos la respuesta) y, lo más importante a nivel de ciberseguridad, se restringe la recursión abierta para evitar que el servidor sea utilizado en ataques de amplificación DNS (DDoS).
```text
options {
    directory "/var/cache/bind";
    recursion yes;
    allow-query { localhost; 192.168.122.0/24; }; # Solo respondemos a nuestra LAN
    forwarders {
        8.8.8.8;
        1.1.1.1;
    };
    dnssec-validation auto;
};
```

#### B. Declaración de Zonas (`named.conf.local`)
Se le indica al servidor que es "Maestro" (autoridad absoluta) sobre el dominio interno y su respectivo bloque de red.
```text
// Zona Directa
zone "aula1.lan" {
    type master;
    file "/etc/bind/db.aula1.lan";
};

// Zona Inversa (Red 192.168.122.x)
zone "122.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/db.192.168.122";
};
```

#### C. Bases de Datos de Zona (Registros DNS)
Se mapea de forma estricta la topología de la red. Cada archivo de zona comienza con el encabezado **SOA** (Start of Authority) indicando el número de serie y los tiempos de refresco.

**Ejemplo de Registros Directos (`db.aula1.lan`):**
```text
@   IN  NS      ns1.aula1.lan.
ns1 IN  A       192.168.122.20
fw  IN  A       192.168.122.1
db  IN  A       192.168.122.45
www IN  A       192.168.122.100
```

**Ejemplo de Registros Inversos (`db.192.168.122`):**
```text
@   IN  NS      ns1.aula1.lan.
20  IN  PTR     ns1.aula1.lan.
1   IN  PTR     fw.aula1.lan.
45  IN  PTR     db.aula1.lan.
100 IN  PTR     www.aula1.lan.
```
*(Nota operativa: En los registros PTR solo se coloca el último octeto de la IP, ya que el motor BIND autocompleta el resto basándose en el nombre de la zona. Además, el FQDN finaliza obligatoriamente con un punto `.` para evitar que se añada el dominio local nuevamente).*

---

### 4. Auditoría Sintáctica y Puesta en Marcha
Nunca se debe reiniciar un servicio crítico en producción sin antes validar la integridad de sus archivos. BIND incluye herramientas nativas para este fin:

1. **Chequeo de Configuración General:**
   ```bash
   sudo named-checkconf
   ```
2. **Chequeo de Integridad de Zonas:**
   ```bash
   sudo named-checkzone aula1.lan /etc/bind/db.aula1.lan
   sudo named-checkzone 122.168.192.in-addr.arpa /etc/bind/db.192.168.122
   ```

Una vez que la validación retorna un estado `OK`, se recarga el demonio de forma segura:
```bash
sudo systemctl restart bind9
```

---

### 5. Forense de Red y Troubleshooting
Para probar empíricamente que la resolución de nombres funciona (tanto desde la caché como desde nuestras bases de datos locales), se utiliza la herramienta `dig`.

```bash
# Prueba de resolución directa (Preguntando por el servidor web)
dig @192.168.122.20 www.aula1.lan

# Prueba de resolución inversa (Preguntando a quién pertenece la IP)
dig @192.168.122.20 -x 192.168.122.100
```
*Si la configuración es exitosa, la salida del comando mostrará el apartado `ANSWER SECTION` devolviendo la IP o el nombre de dominio correspondiente, con un tiempo de respuesta (Query time) cercano a los 0 msec, confirmando la correcta resolución local y evitando fugas de peticiones hacia Internet.*

---

> **Nota:** El desarrollo paso a paso del laboratorio, las capturas de pantalla de las terminales probando la resolución DNS y la aplicación de las políticas de AppArmor se encuentran documentadas en detalle en el archivo `.pdf` adjunto en esta misma carpeta.

***

**Bibliografía:**
* Documentación Oficial de BIND9 (ISC): https://www.isc.org/bind/
* Manual de Servidor DNS de Ubuntu: https://ubuntu.com/server/docs/service-domain-name-service-dns
