# 📂 08 - Configuración de Servicios: SSH, Hardening y Túneles de Red

En el día a día, SSH (*Secure Shell*) no sirve solamente para abrir una terminal a distancia; es la base de toda administración segura en la red. 

Aunque SSH cifra el canal de comunicación (dejando de usar protocolos viejos como Telnet que mandaban todo en texto plano), su configuración base no alcanza para exponer un servidor a Internet sin riesgos. En este laboratorio documentamos las políticas de *Hardening* necesarias y vemos cómo usar SSH para armar tuneles de red qe nos permitan saltar restricciones de firewall.

---

## 1. Hardening de SSH

En distribuciones como Debian, Ubuntu Server o RHEL, el paquete `openssh-server` suele venir preinstalado o se instala con un simple comando. El problema está en dejar la configuración predeterminada como viene.

El puerto 22 es el más escaneado de todo Internet. Si exponés un equipo a la nube sin ajustar nada, en minutos los bots van a empezar a atacarlo por fuerza bruta.

### Configurando `/etc/ssh/sshd_config`
Para asegurar el servicio, tenemos que editar el archivo de configuración principal:

```bash
sudo vim /etc/ssh/sshd_config
```

**Parámetros clave que hay que cambiar:**

1.  **Bloquear el acceso remoto de Root:**
    * *Línea:* `PermitRootLogin no` (Suele venir en `yes` o `prohibit-password`).
    * *Por qué hacerlo:* El usuario `root` existe en todas las máquinas. Si dejamos que se conecte por SSH, a un atacante solo le falta adivinar la contraseña. Lo mejor es obligar a entrar con un usuario estándar y luego usar `sudo`.
2.  **Deshabilitar el inicio de sesión con contraseña:**
    * *Línea:* `PasswordAuthentication no`
    * *Por qué hacerlo:* Fuerza el uso de claves SSH (pública y privada). Esto elimina por completo la posibilidad de que un ataque de fuerza bruta tenga éxito.
3.  **Cambiar el puerto por defecto:**
    * *Línea:* `Port 2244` (o cualquier otro nmero alto).
    * *Por qué hacerlo:* Si bien no frena a un atacante que se dedique específicamente a buscarte, te saca de encima todo el ruido de fondo de los escáneres automáticos que buscan puertos 22 abiertos por la red.

> *Nota: Acordate de reiniciar el servicio cada vez que toques este archivo para que tome los cambios: `sudo systemctl restart sshd`.*

---

## 2. Autenticación con Llaves SSH

Si vamos a desactivar el acceso por contraseña, el usuario va a necesitar un par de llaves criptográficas:

1.  **Crear las llaves (desde tu PC):**
    ```bash
    ssh-keygen -t ed25519 -C "admin@empresa.com"
    ```
    *Esto te genera una llave privada y una publica.*
2.  **Enviar la llave pública al servidor:**
    ```bash
    ssh-copy-id usuario@ip_del_servidor
    ```
    *Este comando copia automaticamente tu llave pública y la mete dentro del archivo `~/.ssh/authorized_keys` del servidor remoto.*

---

## 3. Laboratorio: Túneles SSH (Local Port Forwarding)

SSH tiene herramientas de enrutamiento muy interesantes. Te permite meter otros protocolos (como HTTP o bases de datos) adentro de su túnel seguro. Esto te salva la vida cuando querés acceder a un servicio interno que no tiene salida a Internet por culpa del firewall, pero el puerto SSH sí está abierto.

**Escenario de prueba:**
* **Máquina Virtual (Servidor):** Tenemos un servidor web de prueba corriendo internamente (Python en el puerto 8080). No se puede entrar directamente desde afuera. Su IP es `192.168.122.20`.
* **Maquina Física (Tu PC):** Queremos acceder a ese servidor web interno a través del túnel SSH.

### Paso 1: Levantar el servicio interno (En la VM)
Para hacer la prueba, iniciamos un servidor web rápido en la máquina virtual:
```bash
python3 -m http.server 8080
```

### Paso 2: Abrir el túnel (En tu PC física)
Desde tu terminal, tiramos este comando para hacer el reenvío de puertos (*Local Port Forwarding*):
```bash
ssh -L 8080:127.0.0.1:8080 usuario@tuIP -N
```

**Desglosando el comando:**
* `-L 8080:` Le dice a tu cliente SSH que abra el puerto 8080 en tu red local (localhost).
* `:127.0.0.1:8080` Es hacia dónde va a apuntar el servidor. Básicamente: "Todo lo que entre por este túnel, tiralo al puerto 8080 local del servidor de destino".
* `usuario@tuIP` Es la conexión de entrada (el servidor remoto).
* `-N` Sirve para mantener el túnel abierto en silencio sin abrirte una consola para ejecutar comandos.

### Paso 3: Probar el acceso
Abrí el navegador en **tu máquina física** y entrá a `http://localhost:8080`. Si todo salió bien, vas a ver los directorios de la máquina virtual.

**Qué pasó a nivel red?** Tu navegadr se conectó al puerto 8080 de tu propia PC. El cliente SSH agarró esa conexión, la cifró, la pasó por el puerto 22, cruzó al servidor, se descifró y llegó al proceso de Python como si la solicitud se hubiera hecho desde la propia máquina virtual.

```
> **Nota:** El desarrollo paso a paso del laboratorio, las capturas de pantalla de las terminales probando SSH y la aplicación de local port forwarding se encuentran documentadas en detalle en el archivo `.pdf` adjunto en esta misma carpeta.
