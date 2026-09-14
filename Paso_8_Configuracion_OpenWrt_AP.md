# Manual de Configuración de AP Virtual OpenWrt (Paso 8)

Este documento especifica los pasos detallados para la obtención, importación, implementación y configuración de un Access Point (AP) virtualizado ejecutando **OpenWrt** en el entorno de simulación GNS3. La configuración establece un punto de acceso operando como un **puente transparente de Capa 2 (L2 Ethernet Bridge)** conectado al router principal MikroTik RouterOS (R-CORE).

---

## 1. Resumen de Parámetros de Red

| Parámetro | Valor de Configuración | Descripción |
| :--- | :--- | :--- |
| **Dirección IP de Gestión** | `192.168.88.2/24` | IP estática asignada al AP para administración L3. |
| **Puerta de Enlace (Gateway)** | `192.168.88.1` | Interfaz bridge/Hotspot de MikroTik (R-CORE). |
| **Interfaces en Bridge (L2)** | `eth0`, `eth1` | Puertos integrados al puente `br-lan`. |
| **Servicios Internos (DHCP / NAT)** | Deshabilitados | Se delega el control de direcciones y enrutamiento a MikroTik. |
| **SSID Institucional** | `LosRobles_WiFi` | Identificador de la red inalámbrica (Perfil de Capa Física). |
| **Seguridad Inalámbrica** | WPA3-SAE (SAE) | Estándar obligatorio para el despliegue en hardware real. |

---

## 2. Descarga e Importación de la Imagen OpenWrt en GNS3

### 2.1 Descarga de la Imagen Oficial
1. Acceda al repositorio oficial de descargas de OpenWrt (`downloads.openwrt.org`).
2. Navegue a la arquitectura objetivo: `targets/x86/64/`.
3. Descargue la imagen en formato **Combined EXT4** para QEMU/KVM:
   * **Archivo:** `openwrt-x86-64-generic-ext4-combined.img.gz`
4. Descomprima el archivo `.gz` mediante 7-Zip o utilitario equivalente hasta obtener el archivo ejecutable de disco sin comprimir (`.img`).

### 2.2 Importación de la Plantilla en GNS3
1. **Creación del Appliance:**
   * Abra GNS3 y diríjase a **Edit -> Preferences -> QEMU VMs -> New**.
   * **Name:** `OpenWrt-AP`.
   * **RAM:** `256 MB` (optimizado para consumo mínimo de recursos).
2. **Asignación del Disco Virtual:**
   * Seleccione **New Image** y busque el archivo `.img` descomprimido previamente.
   * Permita que GNS3 copie la imagen al directorio por defecto de imágenes QEMU.
3. **Configuración de Adaptadores de Red:**
   * Seleccione la VM creada y haga clic en **Edit**.
   * En la pestaña **Network**, configure **Adapters: `2`** (mínimo indispensable para operar como bridge L2).
   * Configure el formato de nombres a `eth{0}` y el tipo de tarjeta a **VirtIO (paravirtualized)**.

---

## 3. Asignación de Puertos Físicos en la Topología

* **`eth0`**: Conexión hacia el Router Principal (MikroTik R-CORE / Interfaz Hotspot).
* **`eth1`**: Conexión hacia los clientes finales (`webterm-1` o VM de prueba Ubuntu / Windows).

---

## 4. Procedimientos de Configuración de Red

### 4.1 Configuración mediante Consola CLI (UCI)

Ejecute el siguiente bloque de comandos directamente en la terminal de OpenWrt para establecer la IP estática, apagar los servicios de NAT/DHCP locales y unificar los puertos en el puente transparente:

```bash
# 1. Configurar IP de gestión estática, máscara y puerta de enlace
uci set network.lan.proto='static'
uci set network.lan.ipaddr='192.168.88.2'
uci set network.lan.netmask='255.255.255.0'
uci set network.lan.gateway='192.168.88.1'
uci set network.lan.dns='192.168.88.1'

# 2. Desactivar Servidor DHCP y Firewall interno para evitar doble NAT/DHCP
uci set dhcp.lan.ignore='1'
uci commit dhcp
/etc/init.d/dnsmasq stop 2>/dev/null
/etc/init.d/dnsmasq disable 2>/dev/null
/etc/init.d/firewall stop 2>/dev/null
/etc/init.d/firewall disable 2>/dev/null

# 3. Crear el puente transparente (Bridge) integrando eth0 y eth1
uci set network.lan.device='br-lan'
uci set network.lan.ports='eth0 eth1'
uci commit network

# 4. Reiniciar la pila de red para aplicar los cambios de forma permanente
/etc/init.d/network restart
```

---

### 4.2 Configuración mediante Interfaz Gráfica Web (LuCI)

Para realizar la configuración mediante el entorno gráfico web de OpenWrt (LuCI), siga el procedimiento detallado paso a paso:

1. **Acceso a la Interfaz Web:**
   * Inicie un navegador en un equipo cliente conectado a la red de gestión e ingrese la dirección por defecto: `http://192.168.1.1` (o `http://192.168.88.2` tras la asignación inicial).
   * Inicie sesión con la cuenta de administración (`root`).

2. **Configuración de la Interfaz LAN y Puente (Bridge):**
   * Diríjase al menú **Network -> Interfaces**.
   * Localice la interfaz **LAN** y haga clic en **Edit**.
   * En la pestaña **General Setup**:
     * **Protocol:** Seleccione `Static address`.
     * **IPv4 address:** Asigne `192.168.88.2`.
     * **IPv4 netmask:** Asigne `255.255.255.0`.
     * **IPv4 gateway:** Indique `192.168.88.1`.
     * **Use custom DNS servers:** Ingrese `192.168.88.1`.
   * En la pestaña **Physical Settings** (o **Device Configuration**):
     * Active la casilla **Bridge interfaces** (para crear `br-lan`).
     * Seleccione e integre los puertos físicos **`eth0`** y **`eth1`** dentro del mismo puente.

3. **Deshabilitación de Servicios Locales (DHCP y Firewall):**
   * En la misma ventana de edición de la interfaz LAN, navegue a la sección **DHCP Server**:
     * Marque la casilla **Ignore interface** (Disable DHCP for this interface).
   * En el menú **Network -> Firewall**:
     * Deshabilite las reglas de NAT / Masquerading y detenga la zonificación del firewall local para permitir el tránsito directo de Layer 2 hacia MikroTik.

4. **Aplicación de Cambios:**
   * Haga clic en **Save & Apply** para consolidar los cambios en el sistema.

---

## 5. Especificación Inalámbrica (WPA3-SAE)

En caso de disponer de soporte de radio o al configurar los parámetros por archivo de comandos UCI o desde LuCI (menú **Network -> Wireless**), aplique:

```bash
# Parámetros del Radio Inalámbrico
uci set wireless.radio0.disabled='0'
uci set wireless.radio0.channel='auto'
uci set wireless.radio0.htmode='HT40'

# Parámetros del SSID y Seguridad WPA3-SAE
uci set wireless.default_radio0.ssid='LosRobles_WiFi'
uci set wireless.default_radio0.network='lan'
uci set wireless.default_radio0.mode='ap'
uci set wireless.default_radio0.encryption='sae'
uci set wireless.default_radio0.key='CLAVE_ROBUSTA_INSTITUCIONAL'
uci commit wireless
```

> **Nota de Documentación para el Entorno Virtual (GNS3 / QEMU):**  
> Si la plantilla emulada de OpenWrt en GNS3 opera sin un controlador de radio inalámbrico físico (`radio0`), el tráfico de los clientes se transporta directamente a través del puente Ethernet de Capa 2 (`br-lan`). La especificación **WPA3-SAE (SAE)** queda formalizada como un requisito obligatorio de seguridad para la fase de implementación sobre hardware físico en producción.

---

## 6. Verificación y Validación

### Comprobación del Puente L2 en OpenWrt
Ejecute el comando para confirmar que las interfaces `eth0` y `eth1` pertenecen al puente `br-lan`:

```bash
brctl show
```

**Salida esperada:**
```text
bridge name    bridge id            STP enabled    interfaces
br-lan         7fff.0c9a3b2e0000    no             eth0
                                                   eth1
```

### Configuración requerida en MikroTik (WinBox)
Para asegurar la administración permanente del AP OpenWrt sin interrupciones por el portal cautivo:
1. **Tabla ARP (`IP -> ARP`):** Agregar IP `192.168.88.2` con MAC `0c:de:c4:9e:00:00` en la interfaz del bridge.
2. **IP Bindings del Hotspot (`IP -> Hotspot -> IP Bindings`):** Crear una regla con MAC `0c:de:c4:9e:00:00` y Address `192.168.88.2` con tipo **`bypassed`**.
