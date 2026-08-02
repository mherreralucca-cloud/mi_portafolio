# 📂 09 - Servidor de Archivos Seguro: SFTP Enjaulado (Chroot Jail)

En la administración de servicios, transferir archivos entre clientes y servidores es una tarea de todos los días. Pero, usar protocolos viejos como FTP es un riesgo grave. En este laboratorio reemplazamos FTP por SFTP (*SSH File Transfer Protocol*), aplicando el principio de menor privilegio mediante una "jaula" (*Chroot Jail*) para que los usuarios externos puedan subir archivos de forma cifrada sin poder navegar por el sistema ni abrir una terminal.

---

## 1. Por qué FTP ya no es viable

Aunque antes se enseñaba a levantar servidores FTP con programas como `vsftpd` o `proftpd`, hoy en día usar FTP para transferir datos autenticados es mala práctica por dos razones estructurales:

1. **Tráfico en texto plano:** FTP se diseñó en 1971, cuando la seguridad en redes no era prioridad. Las credenciales (comandos `USER` y `PASS`) y los archivos viajan sin cifrar. Cualquier persona con un sniffer de paquetes (como Wireshark) en la misma red local puede capturar usuario y contraseña sin esfuerzo.
2. **Conflicto con Firewalls y NAT (Modos Activo y Pasivo):** FTP requiere abrir dos conexiones TCP separadas: una para el canal de control (puerto 21) y otra para el canal de datos.
   * *Modo Activo:* El servidor intenta iniciar la conexión de vuelta hacia el cliente. En redes actuales esto falla casi siempre porque los routers y firewalls del cliente bloquean esa conexión entrante.
   * *Modo Pasivo:* Obliga a abrir un rango entero de puertos altos no privilegiados en el servidor, complicando las reglas del firewall.

---

##  2. La Solución: SFTP sobre SSH

SFTP no es FTP corriendo sobre SSL; es un subsistema completo que corre sobre SSH. 

* **Un solo puerto:** Corre por defecto sobre el puerto de SSH (22 o el puerto alternativo que hayamos configurado).
* **Cifrado total:** Todo el canal (autenticación y archivos) viaja encriptado.
* **Amigable con el firewall:** Al usar una sola conexión bidireccional, no genera problemas de enrutamiento ni exige abrir rangos de puertos dinámicos.

---

## 3. Paso a Paso: Creación de Usuarios y Estructura de la Jaula

Para aplicar Confianza Cero (*Zero Trust*), vamos a crear un grupo específico para usuarios de transferencia y preparar el directorio que servirá como jaula.

### Paso 1: Crear grupo y usuario sin acceso a Shell
Creación del grupo y agregado del usuario restringido sin acceso a consola interactiva (`/bin/false` o `/usr/sbin/nologin`):

```bash
sudo groupadd sftp_users
sudo useradd -m -g sftp_users -s /bin/false cliente_web
sudo passwd cliente_web
```

### Paso 2: Crear el directorio de la jaula y ajustar permisos
> **!!! Regla estricta de SSH Chroot:** Para que la jaula funcione y el servidor no rechace la conexión, el directorio raíz de la jaula (`/var/sftp/cliente_web`) **debe pertenecer si o si a `root:root`** y no puede tener permisos de escritura para el grupo ni para otros (`chmod 755`).

```bash
sudo mkdir -p /var/sftp/cliente_web/subidas
sudo chown root:root /var/sftp/cliente_web
sudo chmod 755 /var/sftp/cliente_web
```

Como la carpeta raíz de la jaula es de lectura para el usuario, le creamos una subcarpeta adentro donde sí tenga permisos de escritura para subir sus archivos:

```bash
sudo chown cliente_web:sftp_users /var/sftp/cliente_web/subidas
```

---

## 4. Configuración de OpenSSH (`/etc/ssh/sshd_config`)

Modificamos el archivo de configuración de SSH para indicar que debe hacer el servidor cuando se conecta alguien del grupo `sftp_users`.

```bash
sudo vim /etc/ssh/sshd_config
```

modificamos las siguientes líneas al final del archivo:

```text
# Habilitamos el subsistema interno de SFTP y comentamos el otro Subsystem
Subsystem sftp internal-sftp

# Agregamos la regla al final del archivo. Regla aplicada solo al grupo de transferencias
Match Group sftp_users
    ChrootDirectory /var/sftp/%u
    ForceCommand internal-sftp
    AllowTcpForwarding no
    X11Forwarding no
```

**Explicación de las directivas:**
* `internal-sftp`: Usa el código de SFTP integrado en el propio demonio SSH en lugar de ejecutar un proceso externo, lo que facilita el trabajo dentro del chroot.
* `ChrootDirectory /var/sftp/%u`: Enjaula al usuario en su carpeta correspondiente (`%u` se reemplaza por el nombre del usuario, en este caso `cliente_web`).
* `ForceCommand internal-sftp`: Impide que el usuario ejecute cualquier comando de consola; solo le permite transferir archivos.
* `AllowTcpForwarding no` y `X11Forwarding no`: Bloquean el reenvío de puertos y gráficos para cerrar cualquier posible vía de escape o túnel no autorizado.

Reiniciamos el servicio para aplicar los cambios:
```bash
sudo systemctl restart sshd
```

---

## 5. Pruebas y Validación del Servicio

Probar la configuración desde una máquina cliente o terminal externa:

### Prueba A: Intento de acceso SSH (Debe ser rechazado)
```bash
ssh cliente_web@192.168.122.20
```
*Resultado esperado:* El servidor cierra la conexión inmediatamente, porqe `ForceCommand` y la shell `/bin/false` bloquean el acceso a la terminal.

### Prueba B: Conexión SFTP y verificación de la jaula
```bash
sftp cliente_web@192.168.122.20
```

Una vez iniciada la sesión, verificamos la ruta actual:
```text
sftp> pwd
Remote working directory: /
```
*Resultado que esperamos:* El comando `pwd` devuelve `/` (la raíz), demostrando que la jaula funciona y que el usuario no puede ver ni navegar por los directorios reales del servidor (como `/etc`, `/var/log` o `/home`).

### Prueba C: Subida de archivos
```text
sftp> cd subidas
sftp> put /etc/hosts
Uploading /etc/hosts to /subidas/hosts
sftp> ls
hosts
```
El archivo se transfiere de forma cifrada usando un único puerto y queda guardado dentro de la carpeta `/subidas`.

> **Nota:** El desarrollo paso a paso del laboratorio, las capturas de pantalla de las terminales probando SFTP y la aplicación de Jail Chroot y el hardening se encuentran documentadas en detalle en el archivo `.pdf` adjunto en esta misma carpeta.
