# Configuración de Servidor DNS con BIND9

**Objetivo del Laboratorio:** Implementar y administrar un servidor de nombres de dominio (DNS) utilizando BIND9 en Ubuntu Server. El objetivo abarca desde la configuración inicial como servidor caché (para agilizar peticiones almacenando temporalmente los registros para futuras solicitudes) hasta su despliegue como servidor maestro primario de una zona directa e inversa propia, asumiendo autoridad sobre la resolución de nombres críticos (como www, ftp, mail, etc.) apuntando a la IP privada del entorno.

### Instalación
Para que el servidor DNS funcione, se requieren los paquetes del servicio central junto con utilidades de diagnóstico de red. 

```bash
sudo apt update
sudo apt install bind9 dnsutils net-tools -y
```

**Configuración del Firewall:**
Es crítico habilitar el paso de tráfico de resolución a través del cortafuegos del servidor con este comando:
```bash
sudo ufw allow bind9
```

###  Archivos de Configuración Clave
La estructura de directorios de BIND9 concentra sus configuraciones principales dentro de `/etc/bind/`:

* **Configuración de Caché y Forwarders:**
    *Ruta:* `/etc/bind/named.conf.options` 
    *Detalle:* En este archivo se configura el comportamiento de la caché y se establecen los "forwarders", indicando las direcciones IP de servidores externos (como la IP principal de Google) para delegar las consultas que no se pueden resolver localmente.
* **Definición de Zonas Locales:**
    *Ruta:* `/etc/bind/named.conf.local` 
    *Detalle:* Archivo donde se declara el servidor como "maestro primario" para una zona propia (ej. `midominio.com.ar`) y se especifica la zona de resolución inversa.
* **Archivos de Base de Datos de Zona:**
    *Detalle:* Son los archivos donde se realiza el mapeo estricto de subdominios (`www`, `ftp`, `mail (MX)`, `imap`, `pop3`, `proxy`) hacia las direcciones IP correspondientes.

###  Gestión del Servicio
Para aplicar configuraciones y asegurar la estabilidad del entorno, se utilizan los comandos de gestión del servicio y de auditoría sintáctica propios de BIND9:

* **Auditoría Preventiva de Sintaxis:**
    Antes de cualquier reinicio, es una buena práctica comprobar que no existan errores de tipeo que impidan el arranque del servicio.
    * Comprobar sintaxis del archivo general: `sudo named-checkconf`
    * Comprobar la validez de los archivos de zona: `sudo named-checkzone midominio.com.ar /ruta/al/archivo/db.midominio`
    * 
* **Reiniciar el Demonio:**
    ```bash
    sudo systemctl restart bind9
    ```

###  Pruebas y Verificación
La validación del servicio se realiza desde la línea de comandos para garantizar que la resolución bidireccional y el caché funcionen de manera óptima:

1. **Prueba de Caché y Escucha:** Se utiliza el comando `dig` para comprobar que el servicio efectivamente está escuchando en el puerto 53. Si se realiza un ping (o `dig`) a un dominio externo dos veces, se puede verificar que la segunda consulta tiene un tiempo de espera drásticamente menor, confirmando que la respuesta se sirvió desde la memoria caché.
2. **Resolución de Zona Propia:** Se emplean herramientas como `nslookup` o `resolvectl` para confirmar que el servidor está traduciendo adecuadamente los nombres de nuestra red interna hacia sus respectivas IPs locales.

***

**Bibliografía:**
* TCP/IP Illustrated, Volume 1, The Protocols. W. Richard Stevens. Edición 1. Capítulo 14.
* DNS howto. http://www.tldp.org/HOWTO/DNS-HOWTO.html 
* Manual Ubuntu Server 
