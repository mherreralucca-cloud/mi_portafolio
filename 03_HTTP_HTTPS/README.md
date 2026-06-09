# Despliegue de Servidor Web (HTTP) y Hardening con SSL/TLS (HTTPS)

**Objetivo:** Implementar, configurar y asegurar un servidor web de alto rendimiento utilizando Apache2 en un entorno Ubuntu Server. La práctica abarca desde la configuración inicial de Virtual Hosts para servir múltiples sitios bajo una misma dirección IP, hasta la implementación de controles de acceso perimetral mediante Autenticación Básica (Basic Auth). Finalmente, el laboratorio culmina con el aseguramiento criptográfico del tráfico, habilitando el protocolo HTTPS y vinculando los certificados SSL/TLS generados previamente.

---

###  Marco Teórico y Conceptos Aplicados
* **Virtual Hosts (VHosts):** Capacidad del servidor web para alojar múltiples dominios en una sola máquina y resolver la petición correcta basándose en la cabecera HTTP `Host` del cliente.
* **Autenticación Básica (Basic Auth):** Mecanismo del protocolo HTTP que restringe el acceso a un directorio web, solicitando credenciales (usuario y contraseña cifrada) antes de servir el contenido.
* **Criptografía Asimétrica (SSL/TLS):** Integración del motor de OpenSSL en Apache para cifrar el tráfico bidireccional entre el cliente y el servidor web, garantizando confidencialidad e integridad.

---

###  1. Instalación y Preparación del Entorno
Para este despliegue, se requiere el motor principal del servidor web, las utilidades de Apache (para la gestión de contraseñas) y herramientas de testeo en consola:

```bash
sudo apt update
sudo apt install apache2 apache2-utils links -y
```

**Configuración del Firewall (UFW):**
Es mandatorio habilitar el tráfico entrante en los puertos estándar web (TCP 80 para HTTP y TCP 443 para HTTPS):
```bash
sudo ufw allow "Apache Full"
```

---

###  2. Arquitectura de Directorios y Archivos Clave
Apache en sistemas basados en Debian/Ubuntu utiliza una estructura altamente modular para activar o desactivar componentes mediante enlaces simbólicos:

* **Directorio Raíz Web (`DocumentRoot`):**
    *Ruta:* `/var/www/html/`
    *Detalle:* Ubicación predeterminada donde se alojan los archivos estáticos y dinámicos del sitio web (`index.html`).
* **Sitios Disponibles vs. Habilitados:**
    *Ruta:* `/etc/apache2/sites-available/` y `/etc/apache2/sites-enabled/`
    *Detalle:* En `sites-available` se crean los archivos `.conf` de cada sitio. El comando `a2ensite` crea un enlace simbólico hacia `sites-enabled` para poner el sitio en producción.
* **Módulos Disponibles vs. Habilitados:**
    *Ruta:* `/etc/apache2/mods-enabled/`
    *Detalle:* Controla qué funcionalidades extra están activas en el núcleo de Apache (ej. `ssl`, `rewrite`, `auth_basic`).

---

### 3. Despliegue y Aseguramiento (Hardening)

#### A. Configuración de Sitio y Autenticación Básica
Para restringir el acceso a directorios críticos del servidor web, se generó un archivo de credenciales cifradas utilizando `htpasswd`. 

```bash
# Creación del archivo de contraseñas y asignación de usuario administrador
sudo htpasswd -c /etc/apache2/.htpasswd usuario_admin
```
*En el archivo `.conf` del Virtual Host se configuran las directivas `AuthType Basic`, `AuthName` y `Require valid-user` apuntando a esta ruta para proteger el entorno.*

#### B. Implementación de HTTPS (Módulo SSL)
Para transicionar de un entorno de texto plano (HTTP) a uno cifrado (HTTPS), se habilitó el motor SSL y se activó el Virtual Host predeterminado para el puerto 443.

```bash
# Habilitar el módulo de criptografía en Apache
sudo a2enmod ssl

# Habilitar el Virtual Host seguro
sudo a2ensite default-ssl.conf
```

Dentro del archivo `default-ssl.conf`, se modificaron las directivas clave para apuntar a la bóveda criptográfica del servidor (creada en el laboratorio de OpenSSL):
* `SSLEngine on`
* `SSLCertificateFile      /etc/ssl/certs/server.crt`
* `SSLCertificateKeyFile   /etc/ssl/private/server.key`

#### C. Aplicación de Cambios
Se valida la sintaxis de la configuración para evitar caídas del servicio en producción y se reinicia el demonio:
```bash
sudo apache2ctl configtest
sudo systemctl restart apache2
```

---

###  4. Pruebas, Auditoría y Verificación
La validación se realiza en múltiples fases para confirmar disponibilidad, control de acceso y cifrado:

1. **Prueba Local de Disponibilidad (HTTP):**
    Uso de navegadores basados en terminal (`links`) para comprobar que el servicio base responde localmente en el puerto 80.
    ```bash
    links http://localhost
    ```
2. **Auditoría de Control de Acceso:**
    Al ingresar al directorio protegido, el servidor web detiene la carga y emite un código HTTP 401 (Unauthorized), desplegando el *prompt* de ingreso de credenciales.
3. **Auditoría de Criptografía (HTTPS):**
    Desde un navegador cliente (ej. Firefox o Chrome), se accede a `https://<IP-DEL-SERVIDOR>`. Se inspecciona el candado de seguridad para certificar que Apache está presentando la clave pública correcta (RSA) y que la conexión bidireccional se encuentra cifrada de extremo a extremo, demostrando una defensa exitosa contra ataques MITM (Man-in-the-Middle).

***

**Bibliografía:**
* Documentación Oficial de Apache HTTP Server: https://httpd.apache.org/docs/
* Guía de Servidores Web de Ubuntu Server: https://ubuntu.com/server/docs/web-servers-apache
* Gestión de Módulos y Seguridad: https://ubuntu.com/server/docs/how-to-use-apache2-modules
