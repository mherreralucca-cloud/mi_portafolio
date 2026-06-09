# Preparación de Infraestructura SSL/TLS con OpenSSL

**Objetivo:** Dominar las herramientas criptográficas base en entornos GNU/Linux para preparar un servidor antes de interactuar con una Entidad Certificadora (CA) externa o para asegurar el tráfico en redes privadas. La práctica abarca la gestión de criptografía asimétrica mediante la generación de claves privadas RSA, la creación de Solicitudes de Firma de Certificado (CSR) y la emisión de un certificado público autofirmado (Self-Signed) para su posterior integración en servicios web (HTTPS) o de correo (SMTP/IMAP).

### Instalación
La suite de herramientas de administración criptográfica estándar en la industria es `openssl`. En la mayoría de las distribuciones basadas en Debian/Ubuntu Server se encuentra preinstalada, pero se puede verificar, instalar o actualizar utilizando la gestión de paquetes nativa:

```bash
sudo apt update
sudo apt install openssl -y
```

### Archivos de Configuración Clave y Rutas
El sistema operativo Linux implementa una estructura jerárquica estricta para garantizar la confidencialidad y el aislamiento de los componentes criptográficos:

* **Directorio de Certificados Públicos (`.crt` / `.pem`):**
    *Ruta:* `/etc/ssl/certs/`
    *Detalle:* Espacio destinado a almacenar los certificados públicos del servidor y de las entidades de confianza. Es de acceso público para los servicios que requieran presentar la identidad del servidor a los clientes.
* **Directorio de Claves Privadas (`.key`):**
    *Ruta:* `/etc/ssl/private/`
    *Detalle:* Bóveda de seguridad del sistema. Almacena las claves privadas que permiten descifrar las conexiones. Cuenta con permisos altamente restrictivos (legible únicamente por el superusuario `root`).

### Comandos Clave y Ciclo de Vida del Certificado
El despliegue operativo del material criptográfico consta de pasos secuenciales y estandarizados utilizando la terminal:

1. **Generación de la Clave Privada (RSA de 2048 bits):**
    ```bash
    openssl genrsa -out server.key 2048
    ```
2. **Creación de la Solicitud de Firma (CSR):**
    Este archivo condensa la clave pública y los metadatos de la organización (País, Localidad, Organización y el *Common Name* o dominio del servidor). Es el archivo que en entornos productivos reales se remite a autoridades públicas como Let's Encrypt o DigiCert.
    ```bash
    openssl req -new -key server.key -out server.csr
    ```
3. **Emisión del Certificado Autofirmado (Validez por 365 días):**
    Al operar en un entorno de laboratorio aislado, la propia clave privada firma la solicitud para generar un certificado operativo de forma inmediata.
    ```bash
    openssl x509 -req -days 365 -in server.csr -signkey server.key -out server.crt
    ```
4. **Instalación y Hardening en Rutas del Sistema:**
    Se reubican los archivos en los directorios seguros correspondientes para su consumo por parte de los servicios del servidor.
    ```bash
    sudo cp server.crt /etc/ssl/certs/
    sudo cp server.key /etc/ssl/private/
    ```

### Pruebas y Verificación
Para auditar la validez y consistencia del material generado antes de enlazarlo a un servidor web o de correos, se inspeccionan los metadatos binarios del certificado en formato legible:

* **Auditoría e Inspección de Metadatos:**
    ```bash
    openssl x509 -in /etc/ssl/certs/server.crt -text -noout
    ```
    *Resultado esperado:* La consola decodificará el archivo mostrando campos críticos como el periodo de validez (*Validity*), el algoritmo de firma (*Signature Algorithm: sha256WithRSAEncryption*) y la identidad del emisor (*Issuer*) y del sujeto (*Subject*), verificando que correspondan con los datos cargados en el CSR.

***

**Bibliografía:**
* Guía de Certificados y Seguridad de Ubuntu Server. https://ubuntu.com/server/docs/certificates
* Documentación Oficial del Proyecto OpenSSL. https://www.openssl.org/docs/
