# 🐳 Virtualización Nativa y Contenedores con LXC

**Objetivo:** Implementar, aprovisionar y administrar contenedores nativos de Linux (LXC) en un entorno Ubuntu Server. La práctica busca demostrar el dominio de la virtualización a nivel de Sistema Operativo, desplegando infraestructura aislada que comparte el núcleo (kernel) del host. Este enfoque optimiza radicalmente el consumo de recursos de CPU, memoria e I/O en comparación con las máquinas virtuales (VMs) tradicionales basadas en hipervisores de hardware (como VirtualBox o VMware).

---

### Marco Teórico y Arquitectura de LXC
Para comprender el funcionamiento de los contenedores nativos, se aplican los siguientes conceptos de la arquitectura de Linux:

* **Namespaces (Espacios de nombres):** Característica del kernel que aísla los recursos del sistema. Permite que un contenedor tenga su propio árbol de procesos (PID), puntos de montaje, interfaces de red y usuarios, creyendo que es un sistema operativo independiente.
* **Cgroups (Control Groups):** Mecanismo que limita, prioriza y contabiliza el uso de recursos físicos (CPU, RAM, Disco) para que un contenedor no monopolice el servidor anfitrión.
* **Backing Stores:** Método de almacenamiento del sistema de archivos raíz (`rootfs`) del contenedor. Por defecto, LXC utiliza un simple árbol de directorios, pero en entornos de producción puede integrarse con volúmenes lógicos (LVM) o sistemas de archivos avanzados como Btrfs y ZFS.
* **Snapshots (Instantáneas):** Capacidad de congelar el estado exacto de un contenedor en un punto en el tiempo. Es una herramienta crítica para crear puntos de restauración antes de aplicar actualizaciones o despliegues riesgosos.

---

### 1. Auditoría del Kernel e Instalación Base
El despliegue comienza validando que el núcleo del servidor físico soporte las directivas de aislamiento necesarias.

```bash
# Actualización de repositorios e instalación de la suite LXC y plantillas
sudo apt update
sudo apt install lxc lxc-templates net-tools -y

# Auditoría de compatibilidad del Kernel
lxc-checkconfig
```
*Nota:* El comando `lxc-checkconfig` es vital. Evalúa el kernel en tiempo real y debe devolver los parámetros de `Namespaces` y `Cgroups` en estado "enabled". Sin esta validación, el aislamiento criptográfico y de procesos no es posible

---

### 2. Aprovisionamiento y Despliegue del Contenedor
Se automatiza la creación de una instancia asilada utilizando el motor de plantillas de LXC.

```bash
# Descarga y empaquetado del rootfs basado en plantillas de la comunidad
sudo lxc-create -n ubuntu-priv -t download
```
* **Explicación de parametros: `-t download` interactúa de forma dinámica con los repositorios oficiales de Linux Containers. Durante el asistente, se definió la distribución (Ubuntu), la versión de lanzamiento y la arquitectura (`amd64`). 
* **Ruta de almacenamiento:** El contenedor y su sistema de archivos completo se aíslan en la ruta segura del host: `/var/lib/lxc/ubuntu-priv/`.

---

### 3. Gestión del Demonio y Topología de Red
LXC gestiona sus propias interfaces de red virtuales para proveer conectividad a las instancias sin exponerlas directamente a la red física.

```bash
# Inicializar el contenedor en segundo plano
sudo lxc-start -n ubuntu-priv

# Listado avanzado de instancias y auditoría de estado
sudo lxc-ls --fancy
```
* **Análisis de Salida (`--fancy`):** Este comando despliega una tabla de monitoreo fundamental. Se verifica el nombre (`NAME`), el estado operativo (`STATE: RUNNING`) y, de forma crítica, la dirección IP asignada. 
* **Topología de Red (NAT y Bridge):** Por defecto, LXC asocia el contenedor a una interfaz interna (`lxcbr0`) en modo NAT, otorgándole salida a Internet enmascarada a través de la IP del host físico. En topologías más avanzadas, es posible modificar archivos como `netplan` para configurar un puente (`Bridge`) directo a la tarjeta física, permitiendo que el contenedor obtenga una IP del router de la red local.

---

### 4. Inyección de Consola, Pruebas y Verificación
Para administrar el entorno, LXC ofrece una ventaja táctica sobre las VMs convencionales: permite inyectar una consola directamente en el espacio de usuario del contenedor utilizando el kernel, prescindiendo inicialmente de conexiones de red como SSH.

```bash
# Inyectarse en la terminal root del contenedor (Aislamiento de Namespace)
sudo lxc-attach -n ubuntu-priv
```

**Validación de Conectividad (Contexto Guest):**
Una vez modificado el *prompt* al entorno aislado (`root@ubuntu-priv:~#`), se ejecutaron pruebas de ruteo para certificar la conectividad hacia el exterior.
```bash
# Comprobación de resolución DNS y enrutamiento ICMP
ping google.com

# Verificación de repositorios en el entorno virtual
apt list --upgradable
```
*Resultado:* Si los paquetes ICMP fueron respondidos correctamente y la resolución DNS de `google.com` fue exitosa, confirma que el motor NAT de LXC funciona perfectamente.

---

### 5. Perspectiva del Cloud Computing
El dominio de esta tecnología ligera de contenedores es la base arquitectónica sobre la cual operan los principales proveedores de infraestructura en la nube:
1. **Amazon Web Services (AWS):** Líder global en escalabilidad, integrando microservicios y despliegues masivos.
2. **Google Cloud Platform (GCP):** Destacado por su altísima optimización en la orquestación de contenedores a gran escala (Kubernetes / GKE).
3. **Microsoft Azure:** El estándar corporativo, integrando servicios híbridos y virtualización avanzada para entornos empresariales.

***
El paso a paso mas en detalle con sus capturas se encuentra en el .pdf adjunto, gracias por ver!

**Bibliografía:**
* Documentación Oficial de LXC (Linux Containers): https://linuxcontainers.org/
* Guía de Virtualización y Contenedores de Ubuntu Server: https://ubuntu.com/server/docs/containers-lxc
