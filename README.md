# Drivers Nexmon Pre-Compilados para Extracción de CSI en Raspberry Pi (3B+ / 4B)

Este repositorio alberga **drivers pre-compilados** de **Nexmon CSI** y utilidades para las tarjetas **Raspberry Pi 3 Modelo B+ y Raspberry Pi 4 Modelo B** que permiten la **Extracción de la Información del Estado del Canal (CSI)** a través de su interfaz Wi-Fi (chip Broadcom *bcm43455c0*). Estos drivers son esenciales para investigaciones que involucren detección de movimiento, clasificación de actividades o localización basadas en la propagación de la señal inalámbrica.

## Prerrequisitos

* **Hardware:** Raspberry Pi 3B+ o 4B.
* **Tarjeta SD:** Mínimo 16GB.
* **PC Host (para configuración y análisis):** Conexión Ethernet y capacidad para compartir Internet (si se usa la RPi sin Wi-Fi).
* **Cable:** Cable Ethernet.
* **Monitor y teclado**: Se usaran para las configuraciones iniciales.

## Instalación del Sistema Operativo (SO)

La extracción de **CSI** (Channel State Information) a través de la interfaz Wi-Fi (como con el proyecto Nexmon) es sensible a la versión del kernel. Es **crucial** utilizar una imagen del SO con una versión de kernel que sea **compatible** con los drivers pre-compilados de este repositorio.

Recomendamos utilizar una de las siguientes imágenes oficiales de **Raspberry Pi OS Lite (*32-bit* o *armhf*)**, las cuales han demostrado ser compatibles:

Kernel Version | Link
---------------|-----
5.10.92        | <https://downloads.raspberrypi.org/raspios_lite_armhf/images/raspios_lite_armhf-2022-01-28/>
5.4.51         | <https://downloads.raspberrypi.org/raspios_lite_armhf/images/raspios_lite_armhf-2020-08-24/>
4.19.97        | <https://downloads.raspberrypi.org/raspbian_lite/images/raspbian_lite-2020-02-14/>

Usaremos **Raspberry Pi Imager** (o una herramienta alternativa como *Balena Etcher*) para grabar el sistema operativo en la tarjeta SD.

1. **Obtener el Software y la Imagen:**
    * **Descarga** e instala **Raspberry Pi Imager** en tu PC Host.
    * **Descarga la imagen del SO compatible** (consulta la tabla de compatibilidad) en tu PC. No es necesario descomprimir el archivo.

2. **Grabación y Configuración del Sistema Operativo:**
    * Inserta la **tarjeta SD** (mínimo 16GB) en tu PC Host.
    * Abre **Raspberry Pi Imager**.
    * En las opciones de selección de imagen, elige **"Usar personalizado"** y selecciona el archivo de imagen del SO que descargaste.
    * Selecciona tu tarjeta SD como el **dispositivo de almacenamiento** de destino.
    * Haz clic en configuraciones para establecer las **opciones avanzadas**:
        * Habilita **SSH** para el acceso remoto.
        * Establece el **nombre de usuario** en **`csi`** y define una **contraseña segura**.
        * *(Opcional)* Configura la **conexión Wi-Fi** si lo deseas, aunque la configuración inicial se hará por Ethernet.
    * Haz clic en **Escribir** para iniciar el proceso.

3. **Primer Encendido de la Raspberry Pi:**
    * Una vez que el proceso de escritura haya finalizado, retira la tarjeta SD de forma **segura** del PC.
    * Insértala en tu **Raspberry Pi** e inicia el dispositivo.

-----

## Configuraciones Inicial  

### Acceso Remoto Raspberry Pi: Configuración SSH por Ethernet

Para asegurar una conexión remota estable y predecible a través del cable Ethernet, es esencial asignar una **dirección IP estática** a la Raspberry Pi.

Conecta un monitor y un teclado a la Pi, espera a que el SO cargue e inicia sesión con el usuario **`csi`** y la contraseña que estableciste durante la instalación. Y realize los siguientes pasos:

1. **Identificar la Interfaz:** Determina el nombre de la interfaz de red cableada (generalmente `eth0` o similar a `enx...`) con el comando:

    ```bash
    ip addr
    ```

2. **Modificar la Configuración de Red:** Abre el archivo de configuración de **DHCP** para editarlo:

    ```bash
    sudo nano /etc/dhcpcd.conf
    ```

3. **Añadir Configuración Estática:** Dirígete al final del archivo y agrega el siguiente bloque de configuración. Asegúrate de **reemplazar** `[nombre_de_tu_interfaz]` con el nombre que identificaste en el paso 1.

    ```bash
    interface [nombre_de_tu_interfaz]
    static ip_address=192.168.10.2/24
    static routers=192.168.10.1
    static domain_name_servers=192.168.10.1 8.8.8.8
    ```

    *(Esta configuración establece la IP de la Pi en `192.168.10.2`. Asume que la IP de tu PC Host será `192.168.10.1` dentro de la misma subred.)*

4. **Aplicar Cambios:** Guarda el archivo (`Ctrl+O`, `Enter`) y sal (`Ctrl+X`). Luego, reinicia la Raspberry Pi para que la nueva configuración de red entre en vigor:

    ```bash
    sudo reboot
    ```

### Conexión SSH desde el PC Host (Sistema Operativo Ubuntu 24.04.3 LTS)

Para comunicarte con la Raspberry Pi en su IP estática (`192.168.10.2`), tu PC Host con también debe tener una IP estática en el mismo rango (`192.168.10.1`).

### 1. Configurar IP Estática en el PC Host

1. **Abrir Configuración:** Ve a **Configuración** \> **Red** en tu sistema Ubuntu.
2. **Modificar Conexión por Cable:** Localiza tu conexión Ethernet, crea una nueva conexión y haz clic en el ingresa a sus configuraciones.
3. **Configuración IPv4:** Dirígete a la pestaña **"IPv4"**.
4. **Método Manual:** Cambia el **"Método"** a **"Manual"**.
5. **Establecer Direcciones:** En la sección de **Direcciones**, introduce:
      * **Dirección:** `192.168.10.1`
      * **Máscara de red:** `255.255.255.0`
      * **Puerta de enlace:** Déjalo en blanco.
6. Haz clic en **"Aplicar"**. Tu PC Host ahora tiene una dirección IP fija y está listo para comunicarse.

### 2. Verificar Conexión y Solucionar Problemas de Firewall

Ahora que ambas máquinas tienen IPs fijas, prueba la conexión.

Paso | Comando en la Raspberry Pi (con monitor) | Resultado Esperado
:--- | :--------------------------------------- | :-----------------
**Prueba de Conexión** | `ping 192.168.10.1` | Respuesta exitosa (`bytes from...`)

Si recibes un error como **"Destination Host Unreachable"** o no hay respuesta, lo más probable es que el **Firewall (UFW)** de Ubuntu esté bloqueando la comunicación.

### 3. Solución de Problemas (Diagnóstico con `tcpdump`)

Si el ping falla, usa `tcpdump` para diagnosticar si el paquete de la Pi está llegando al PC Host.

1. **Instalar `tcpdump`** (si no está instalado):

    ```bash
    sudo apt install tcpdump
    ```

2. **Identificar Interfaz Ethernet**

    ```bash
    ip addr
    ```

3. **Escuchar Tráfico (en la terminal)** Reemplaza `[nombre_interfaz_ethernet]` con tu interfaz real.

    ```bash
    sudo tcpdump -i [nombre_interfaz_ethernet] icmp
    ```

4. **Ejecutar Ping (en la terminal de la Pi) mientras `tcpdump` escucha:**

    ```bash
    ping 192.168.10.1
    ```

**Interpretación:**

* **Si ves líneas en `tcpdump`:** El paquete SÍ llega. El problema es definitivamente el firewall del PC Host.
* **Si NO ves líneas:** El paquete NO llega. Revisa el cable, la configuración de IP de la Pi o la configuración de IP de PC Host.

### 4. Desactivación Temporal del Firewall

Para confirmar que el firewall es la causa, desactívalo temporalmente en el PC Host y vuelve a probar el ping desde la Pi.

1. **Desactivar UFW (en PC Host):**

    ```bash
    sudo ufw disable
    ```

2. **Volver a intentar Ping (en la Pi):**

    ```bash
    ping 192.168.10.1
    ```

    Si funciona, el firewall era el problema.

### 5. Crear Regla de Confianza

Vuelve a activar el firewall de Ubuntu, pero con una regla que permita el tráfico desde la Pi.

1. **Identificar la Interfaz Ethernet** de nuevo si no lo recuerdas.

    ```bash
    ip addr
    ```

2. **Crear Regla de Permiso (en el PC Host):** Reemplaza `[nombre_interfaz_ethernet]`.

    ```bash
    sudo ufw allow in on [nombre_interfaz_ethernet] from 192.168.10.2
    ```

3. **Activar Firewall:**

    ```bash
    sudo ufw enable
    ```

4. **Recargar Firewall:**

    ```bash
    sudo ufw reload
    ```

Ahora, la conexión SSH (`ssh csi@192.168.10.2`) debería ser permanente y funcionar correctamente.

### Configurar Internet Compartido (PC Host como Router)

Ahora que tienes conectividad local, configura tu PC Host  para que comparta su conexión Wi-Fi con la Raspberry Pi a través del cable Ethernet.

1. **Activar el Reenvío de Paquetes (IP Forwarding):**
    Permite al sistema operativo del PC Host enrutar el tráfico entre sus interfaces (Wi-Fi y Ethernet).

    ```bash
    sudo sysctl -w net.ipv4.ip_forward=1
    ```

2. **Configurar NAT (Network Address Translation):**
    Esto "enmascara" el tráfico saliente de la Pi para que parezca que viene de la interfaz Wi-Fi de tu PC Host. Necesitas los nombres de tu interfaz **Wi-Fi** (`[nombre_interfaz_wifi]`) y **Ethernet**. Utiliza `ip addr` para identificar tu interfaz Wi-Fi (`[nombre_interfaz_wifi]`).

    ```bash
    sudo iptables -t nat -A POSTROUTING -o [nombre_interfaz_wifi] -j MASQUERADE
    ```

3. **Ajustar la Política de Reenvío de UFW:**
    Permite que el firewall reenvíe este tráfico.
      * Edita el archivo de configuración de UFW:

        ```bash
        sudo nano /etc/default/ufw
        ```

      * Busca la línea `DEFAULT_FORWARD_POLICY="DROP"` y cámbiala a:

        ```bash
        DEFAULT_FORWARD_POLICY="ACCEPT"
        ```

      * Guarda el archivo (`Ctrl+X`, `Y`, `Enter`) y recarga el firewall:

        ```bash
        sudo ufw reload
        ```

La Raspberry Pi ya debería tener acceso a Internet a través de la conexión compartida por Ethernet. Puedes verificarlo desde la Pi con el comando `ping 8.8.8.8`.

Aquí tienes la redacción mejorada para la sección de problemas comunes, haciéndola más clara y estructurada para el usuario:

## Solución de Problemas Comunes

### Wi-Fi Bloqueado por `rfkill`

Si al intentar activar o configurar la conexión Wi-Fi recibes un mensaje indicando que **"wifi is currently blocked by rfkill"**, significa que el dispositivo inalámbrico ha sido desactivado por *software* (soft-blocked) en el sistema.

Sigue estos pasos para solucionarlo:

1. **Verificar el Estado del Bloqueo:**
    Usa el comando `rfkill list` para identificar qué dispositivos están bloqueados. Busca tu dispositivo Wi-Fi (generalmente listado como `wlan` o `wlan0`) y verifica si el campo **`Soft blocked`** dice **`yes`**.

2. **Desbloquear el Wi-Fi:**
    Ejecuta el siguiente comando para eliminar el bloqueo de *software* en todos los dispositivos Wi-Fi:

    ```bash
    sudo rfkill unblock wifi
    ```

3. **Configurar la Región Inalámbrica:**
    Es fundamental establecer correctamente la región Wi-Fi, ya que esto afecta los canales y la potencia permitidos, evitando futuros bloqueos por *software*.

      * Ejecuta la herramienta de configuración de la Raspberry Pi:

        ```bash
        sudo raspi-config
        ```

      * Navega a **Localisation Options** (Opciones de Localización).
      * Selecciona **WLAN Country** (País WLAN) y elige tu región geográfica.

Una vez completados estos pasos, el Wi-Fi debería estar desbloqueado y operativo para la configuración.

Aquí tienes una redacción mejorada y más profesional para la nota, asegurando que el usuario comprenda el riesgo y la acción preventiva de manera clara.

-----

## Nota Importante: Preservación del Kernel

Es **crucial evitar la ejecución de `sudo apt upgrade`** ya que este comando podría actualizar el **kernel de Linux**, lo que rompería la compatibilidad con los *drivers* CSI pre-compilados (Nexmon). Al momento de redactar esta guía, la extracción de CSI solo es estable en kernels hasta la **versión 5.10**.

Si necesitas actualizar otros paquetes del sistema sin afectar el kernel, primero debes **bloquear la versión actual del kernel** con los siguientes comandos:

```bash
sudo su
apt-mark hold raspberrypi-kernel
```

Para desbloquear la actualización del kernel en el futuro (si se libera una versión compatible del *driver*), usa:

```bash
sudo apt-mark unhold raspberrypi-kernel
```

-----

## Clonar el Repositorio

Una vez que hayas establecido la conexión SSH con éxito y te encuentres en la terminal de la Raspberry Pi, el siguiente paso es instalar **Git** para clonar este repositorio, el cual contiene los *drivers* y *scripts* necesarios.

1. **Actualizar la Lista de Paquetes:**
    Asegúrate de que la lista de paquetes del sistema esté actualizada (asumiendo que configuraste el internet compartido correctamente).

    ```bash
    sudo apt update
    ```

2. **Instalar Git:**
    Instala la herramienta de control de versiones Git.

    ```bash
    sudo apt install git -y
    ```

3. **Clonar el Repositorio:**
    Navega al directorio donde deseas guardar los archivos (por ejemplo, el directorio `home`) y clona el repositorio:

    ```bash
    cd ~
    git clone [URL]
    cd nombre_del_repositorio
    ```

## Instalar los Controladores Nexmon CSI Pre-Compilados

El repositorio incluye un *script* de instalación (`install.sh`) que automatiza la copia de los *drivers* pre-compilados y las utilidades del kernel a las ubicaciones correctas del sistema.

1. **Ejecutar el Script de Instalación:**
    Asegúrate de estar en el directorio raíz del repositorio (por ejemplo, `/home/csi/nombre_del_repositorio`) y ejecuta el *script* con privilegios de superusuario:

    ```bash
    sudo bash install.sh
    ```

2. **Solución de Errores de Ruta (Si el Usuario es Diferente a `csi`):**
    Si el *script* falla con errores de tipo **"No se puede localizar la carpeta `/home/csi/...`"**, significa que el nombre de usuario de tu Raspberry Pi es diferente a `csi`.

      * **Corrección:** Abre el archivo `install.sh` y modifica manualmente todas las líneas que contengan la ruta `/home/csi/` para usar el nombre de usuario correcto (por ejemplo, `/home/pi/`).

        ```bash
        nano install.sh
        # Presiona Ctrl+\ para buscar y reemplazar si tu editor lo soporta.
        ```

3. **Finalizar la Instalación:**
    Una vez que el *script* haya terminado sin errores, el sistema debe reiniciarse para que los nuevos módulos del kernel se carguen correctamente.

    ```bash
    sudo reboot
    ```

Tras el reinicio, los *drivers* estarán activos y la Raspberry Pi estará lista para la extracción de datos CSI.

El objetivo de esta herramienta es capturar **Información del Estado del Canal (CSI)** de las transmisiones Wi-Fi utilizando *drivers* parcheados (Nexmon) en la Raspberry Pi.

## Guía de Captura de Datos CSI

La herramienta Nexmon CSI captura la información de la capa física (PHY) del Wi-Fi y la encapsula dentro de paquetes **UDP** que son emitidos por la Raspberry Pi.

### Mecanismo de Emisión y Transporte

Parámetro | Valor | Descripción
:--- | :--- | :---
**Protocolo de Transporte** | UDP (User Datagram Protocol) | Se utiliza para el transporte de datos sin conexión.
**Dirección Fuente** | `10.10.10.10` | IP de la Raspberry Pi (ficticia o definida por el *driver*).
**Dirección Destino** | `255.255.255.255` (Broadcast) | Los datos se envían como *broadcast* a la red local.
**Puerto Destino** | `5500` | Puerto UDP específico donde se reciben los paquetes CSI.

**Formato de Archivo:** Los paquetes se pueden capturar utilizando herramientas como **`tcpdump`** y se almacenan en archivos con formato **`.pcap`**. Estos archivos son compatibles con analizadores de red como **Wireshark** o pueden ser procesados directamente por *scripts* de análisis.

### Estructura del Payload CSI (Dentro del Paquete UDP)

El **Payload** de cada paquete UDP contiene la estructura de metadatos y los datos CSI complejos reales. La estructura del *payload* es la siguiente:

Bytes | Tipo de Dato | Descripción
:---: | :---: | :---
**2** | `uint16` | **Magic Bytes (`0x1111`):** Indicador de inicio del *payload* CSI.
**1** | `uint8` | **RSSI:** Intensidad de la señal recibida, en formato de Complemento a Dos.
**2** | `uint16` | **Frame Control Byte:** Byte que indica el tipo de *frame* Wi-Fi que desencadenó la captura.
**6** | `uint8[6]` | **MAC Fuente:** Dirección MAC del dispositivo Wi-Fi que envió el *frame* original.
**2** | `uint16` | **Número de Secuencia:** Número de secuencia del *frame* Wi-Fi.
**2** | `uint16` | **Core y Spatial Stream:** Los 3 bits menos significativos indican el **Core** y los siguientes 3 bits indican el **Spatial Stream** (ej. `0x0019` significa Core 0, Spatial Stream 3).
**2** | `uint16` | **Chanspec:** Especificación del canal utilizada durante la extracción.
**2** | `uint16` | **Chip Version:** Identificador de la versión del chip Broadcom (ej. bcm4339, bcm43455c0).
**Variable** | `int16[]` | **Datos CSI Reales:** Los valores complejos de CSI.

### Formato y Tamaño de los Datos CSI

Los datos CSI reales (`CSI Data`) son el componente más importante del *payload* y su tamaño depende del ancho de banda del canal Wi-Fi:

Ancho de Banda | Subportadoras (Subcarriers) | Longitud del CSI por paquete | Formato de Datos
:---: | :---: | :--- | :---
**20 MHz** | 64 | 64 x 4 bytes | **`int16`** real e imaginario (interleaved)
**40 MHz** | 128 | 128 x 4 bytes | **`int16`** real e imaginario (interleaved)
**80 MHz** | 256 | 256 x 4 bytes | **`int16`** real e imaginario (interleaved)

**Nota sobre el Formato Flotante:** Algunos chips (como el **bcm4358** y **bcm4366c0**) devuelven valores CSI en un formato de punto flotante comprimido, no en el formato simple `int16`. Se requieren *scripts* especializados (como `unpack_float.c` o equivalentes en Python) para decodificar estos valores correctamente.

### Herramientas de Análisis

Los datos capturados pueden ser analizados con:

* **Wireshark:** Útil para inspeccionar los paquetes `.pcap` y verificar la estructura del UDP y los metadatos.
* **Scripts de MATLAB:** Se proporcionan *scripts* de ejemplo (`matlab/csireader.m`) que incluyen la funcionalidad para leer y graficar los datos en formato `int16` y decodificar el formato flotante (requiere compilar `matlab/unpack_float.c`).
* **Scripts de Python:** Un *script* de análisis en Python está en desarrollo para decodificar y procesar los datos capturados.

Aquí tienes una guía clara y paso a paso para el proceso de captura de datos CSI, asumiendo que ya has instalado los controladores Nexmon CSI y tienes acceso SSH por Ethernet.

### Proceso de Captura de Datos CSI

La captura de la Información del Estado del Canal (CSI) implica configurar el chip Wi-Fi para que opere en modo monitor y dirija los datos CSI, encapsulados en paquetes UDP, hacia un puerto de red específico (`5500`).

**Nota Importante:** Durante la captura, el chip Wi-Fi interno estará dedicado a la extracción de CSI y **no podrá utilizarse para conectarse a una red Wi-Fi**. Asegúrate de mantener tu conexión SSH por **Ethernet**.

### 1. Configurar los Parámetros de Extracción

Utiliza la herramienta `mcp` (**makecsiparams**) para generar una cadena de parámetros en Base64 que configure el extractor del chip Wi-Fi.

1. **Generar la Cadena de Configuración:**
    El siguiente ejemplo configura la captura en el **Canal 36** con un **Ancho de Banda de 80 MHz**. Los parámetros `-C 1` (Core) y `-N 1` (Spatial Stream) generalmente no necesitan cambiarse en una Raspberry Pi estándar.

    ```bash
    mcp -C 1 -N 1 -c 36/80
    ```

    **Salida de Ejemplo:** ```KuABEQAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA==```

    *(**Nota:** Si deseas aplicar filtros por MAC o tipo de frame, ejecuta `mcp -h` para ver opciones adicionales).*

2. **Configurar el Chip con `nexutil`:**
    Usa la utilidad `nexutil` junto con la cadena Base64 generada (reemplaza el ejemplo si generaste uno diferente) para inyectar la configuración en el *firmware* del chip Wi-Fi.

    **Asegúrate de que la interfaz wlan0 esté activa:**

    ```bash
    sudo ifconfig wlan0 up
    ```

    **Inyecta los parámetros CSI (longitud 34):**

    ```bash
    sudo nexutil -Iwlan0 -s500 -b -l34 -vKuABEQAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA==
    ```

### 2. Habilitar el Modo Monitor

Para capturar los paquetes, la interfaz necesita estar en modo monitor.

1. **Crear Interfaz de Monitor:**
    Crea una nueva interfaz virtual (`mon0`) de tipo monitor a partir de `wlan0`.

    ```bash
    sudo iw dev wlan0 interface add mon0 type monitor
    ```

2. **Activar la Interfaz:**
    Asegúrate de que la nueva interfaz de monitor esté activa.

    ```bash
    sudo ip link set mon0 up
    ```

### 3. Recolección de Datos CSI con `tcpdump`

Los datos CSI son emitidos como paquetes UDP al puerto `5500`. Utiliza `tcpdump` para interceptar estos paquetes y guardarlos en un archivo.

1. **Captura y Almacenamiento en Archivo (.pcap):**
    Este comando capturará los primeros **1000** paquetes CSI en el puerto `5500` y los guardará en un archivo `.pcap` para su posterior análisis.

    ```bash
    sudo tcpdump -i wlan0 dst port 5500 -vv -w output.pcap -c 1000
    ```

      * **`-i wlan0`**: Especifica la interfaz de monitor.
      * **`dst port 5500`**: Filtra solo los paquetes CSI.
      * **`-w output.pcap`**: Especifica el nombre del archivo de salida.
      * **`-c 1000`**: Limita la captura a 1000 paquetes.

2. **Visualización Rápida (Sin Guardar):**
    Si solo deseas ver el flujo de datos CSI en tiempo real, omite los argumentos `-w` y `-c`.

    ```bash
    sudo tcpdump -i wlan0 dst port 5500
    ```

3. **Captura de Datos por Tiempo Determinado**

    Para capturar datos CSI durante un período de tiempo específico en lugar de por un número fijo de paquetes, la forma más sencilla es utilizar el comando **`timeout`** junto con `tcpdump`.

    Simplemente antepón `timeout [TIEMPO_EN_SEGUNDOS]s` a tu comando `tcpdump` y **elimina la opción `-c`** (conteo de paquetes) para que la captura se detenga exclusivamente por el límite de tiempo.

    ```bash
    sudo timeout [TIEMPO_EN_SEGUNDOS]s tcpdump -i wlan0 dst port 5500 -vv -w output.pcap
    ```

    * **`timeout [TIEMPO_EN_SEGUNDOS]s`**: Indica al sistema que ejecute el comando siguiente y lo detenga automáticamente después del número de segundos especificado.
    * **Eliminación de `-c [número]`**: Esto quita el límite de conteo de paquetes, asegurando que el *timeout* sea el único factor de detención.

    Si deseas capturar datos CSI durante medio minuto, el comando sería:

    ```bash
    sudo timeout 30s tcpdump -i wlan0 dst port 5500 -vv -w captura_30s.pcap
    ```

    El sistema iniciará la captura y la detendrá automáticamente pasados los 30 segundos.

### Transferencia de Archivos

Utiliza **Secure Copy Protocol (SCP)** para transferir los archivos de datos (`.pcap`, `.txt`, etc.) de la Raspberry Pi al PC Host para su análisis.

1. **Desde la Raspberry Pi (Pull desde el Host):**
    Desde tu PC Host, usa `scp` para "tirar" de los archivos:

    **Sintaxis:** `scp <usuario_pi>@<ip_pi>:<ruta_archivo_en_pi> <ruta_local_en_pc>`

    ```bash
    scp CSI@192.168.1.100:/home/csi/csi_raw_*.pcap Descargas/datos_csi/
    ```

2. **Desde la Raspberry Pi (Push al Host):**
    También puedes configurar claves SSH sin contraseña para transferir desde la Pi, pero la opción anterior es la más simple para un entorno de análisis de una sola máquina.

    ¡Claro! Aquí tienes un mensaje de cierre conciso y profesional, junto con sugerencias de nombres para el repositorio, optimizados para SEO.

¡Felicidades! Has completado la configuración de tu sistema de captura de Información del Estado del Canal (**CSI**). Tu Raspberry Pi está lista para transformar las señales Wi-Fi en datos detallados que pueden ser utilizados en proyectos avanzados de detección de presencia, reconocimiento de gestos o monitoreo ambiental.

A partir de ahora, puedes usar las herramientas de captura (`tcpdump`) y los *scripts* de análisis (como los de MATLAB o Python en la carpeta `utils/`) para empezar a recolectar y visualizar tus propios datos CSI.

Si tienes problemas o deseas contribuir, por favor, abre un *issue* en este repositorio.

¡Que disfrutes de tu proyecto!
