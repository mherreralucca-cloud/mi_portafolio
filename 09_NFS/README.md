# 📂 10 - Almacenamiento en Red: NFSv4, Firewall y Autofs

Durante mucho tiempo, NFS (*Network File System*) viene siendo el estándar en servidores Linux para centralizar datos. Sus casos de uso principales son:

* **Clústeres Web (Load Balancing):** Varios servidores web front-end leyendo y escribiendo sobre la misma carpeta de imágenes para mantener la consistencia.
* **Virtualización (Proxmox/VMware):** Almacenar discos duros de las VM. Si un hipervisor físico se quema, otro nodo levanta la VM en segundos porque el disco está en el almacenamiento central.
* **Contenedores (Docker/K8s):** Adjuntar volúmenes persistentes a contenedores que nacen y mueren todo el tiempo.

---

## 1. Arquitectura: La trampa de NFSv3 vs. la seguridad de NFSv4

Durante años NFS fue una complicacion para los administradores de redes debido a cómo manejaba los puertos.

* **El problema de NFSv3:** Depende del servicio *RPCbind*. Cuando un cliente quiere conectarse, el servidor le asigna puertos dinámicos y aleatorios. Intentar asegurar esto en un Firewall implica abrir rangos grandes de puertos, rompiendo cualquier política de seguridad.
* **El estándar NFSv4:** Elimina RPCbind. Todo el tráfico de transferencia y control viaja exclusivamente por un único puerto TCP (2049). Esto nos permite cerrar el firewall por completo y abrir solo esa puerta.

---

## 2. Configuración del Servidor y Control de Privilegios

En el equipo que actúa como servidor, preparamos el directorio que vamos a exportar a la red.

```bash
sudo mkdir -p /srv/nfs/storage_compartido
sudo chown nobody:nogroup /srv/nfs/storage_compartido
sudo chmod 777 /srv/nfs/storage_compartido
```

### El archivo `/etc/exports`
Acá es donde realmente se aplican las reglas de acceso. Editamos el archivo para declarar qué red puede entrar y con qué permisos:

```text
/srv/nfs/storage_compartido tuIP/MASC(rw,sync,no_subtree_check,root_squash)
```

**Desglose de directivas de seguridad:**
* `IP/MASC`: Filtramos por red. Nadie afuera de este segmento puede ver el recurso.
* `sync`: Fuerza a que los datos se escriban físicamente en el disco antes de confirmarle al cliente que la operación terminó (evita pérdida de datos ante cortes de luz).
* `root_squash`: **Regla de seguridad.** Si el usuario `root` de la máquina cliente intenta crear un archivo en el servidor, NFS intercepta la acción y le "aplasta" los privilegios, convirtiéndolo en el usuario sin permisos `nobody`. Esto evita que un servidor comprometido escale privilegios hacia el almacenamiento central.

Aplicamos los cambios y reiniciamos:
```bash
sudo exportfs -a
sudo systemctl restart nfs-kernel-server
```

---

## 3. Cierre del Firewall

Como usamos NFSv4, podemos aplicar una regla de firewall exacta y denegar todo lo demás. Solo permitimos tráfico entrante al puerto 2049 desde nuestra red de confianza:

```bash
sudo ufw allow from 192.168.122.0/24 to any port 2049
sudo ufw enable
```

---

## 4. Configuración del Cliente: La ventaja de Autofs

En el cliente, podríamos montar este disco directamente en `/etc/fstab`. Peroo, esto es un error. Porque si el servidor NFS se reinicia o se corta un switch de red, la máquina cliente se va a quedar congelada en el arranque esperando que el disco aparezca (Timeout).

Para evitar esto usamos **Autofs**. Este daemon intercepta las peticiones del usuario; monta el disco en red *solamente* en el instante en que alguien intenta entrar a la carpeta, y lo desmonta automáticamente si nadie lo usa durante un tiempo determinado (ahorrando ancho de banda y conexiones TCP).

### Configuración de Autofs
Editamos el archivo maestro `/etc/auto.master` para definir dónde van a vivir los montajes:
```text
/- /etc/auto.nfs --timeout=60
```

Creamos el archivo de mapeo `/etc/auto.nfs` indicando la ruta local, el protocolo y la IP del servidor:
```text
/mnt/nfs/storage -fstype=nfs4,rw 192.168.122.20:/srv/nfs/storage_compartido
```

Reiniciamos el servicio en el cliente:
```bash
sudo systemctl restart autofs
```

---

## 5. Troubleshooting (Forense de Errores)

Nota: En un entorno NFSv4 con Firewall, herramientas clásicas como `showmount -e` van a fallar (Timeout) porque intentan usar el puerto 111 de RPC. 

Para probar la conexión real de la arquitectura, siempre hacer un montaje manual antes de configurar Autofs. Si esto funciona, el problema está en la configuración del cliente, no en la red:
```bash
mount -v -t nfs4 IP_SERVIDOR:/ruta_exportada /ruta_local_de_prueba
```

---

## Conclusión: Arquitectura del Laboratorio

En este laboratorio vemos que montar almacenamiento en red no es solo compartir una carpeta. Descartamos la inseguridad de NFSv3, forzamos tráfico por un único puerto (2049) para asegurar el Firewall, achicamos la escalada de privilegios con `root_squash` y optimiizamos los recursos de red del cliente usando montaje de bajo demanda con Autofs.

> **Nota:** El desarrollo paso a paso del laboratorio, las capturas de pantalla de las terminales probando SFTP y la aplicación de Jail Chroot y el hardening se encuentran documentadas en detalle en el archivo `.pdf` adjunto en esta misma carpeta.
