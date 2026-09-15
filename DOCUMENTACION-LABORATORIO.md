# Documentación completa del laboratorio

## Red inalámbrica segura para el Colegio Los Robles

Laboratorio implementado con **GNS3**, **MikroTik CHR RouterOS 7.x**, **OpenWrt x86/64** y un cliente de pruebas. El objetivo es validar la lógica de una red inalámbrica segura antes de trasladarla a equipos físicos.

> **Nota sobre el alcance:** GNS3 permite probar direccionamiento, DHCP, DNS, NAT, firewall, Hotspot, RADIUS, QoS y el puente Ethernet del AP. Una VM OpenWrt conectada mediante interfaces Ethernet no representa una radio Wi-Fi real. La cobertura, interferencia, roaming, capacidad radioeléctrica y WPA3 deben verificarse posteriormente con hardware compatible.

---

## 1. Información general

### 1.1 Objetivo

Diseñar y validar una red inalámbrica administrable que proporcione:

- Conectividad a Internet para los usuarios.
- Separación lógica de funciones de red.
- Autenticación mediante portal cautivo y RADIUS.
- Control de ancho de banda por perfil.
- Protección del router mediante firewall.
- Resolución DNS controlada.
- Registro de eventos para auditoría.
- Un AP OpenWrt funcionando como puente transparente de capa 2.

### 1.2 Roles de los equipos

| Equipo | Tecnología | Función |
|---|---|---|
| `R-CORE` | MikroTik CHR RouterOS 7.x | Gateway, NAT, DHCP, DNS, firewall, Hotspot, RADIUS y QoS |
| `AP-01` | OpenWrt x86/64 | AP virtual y puente Ethernet de capa 2 |
| `CLIENTE-01` | Webterm o VM Linux | Navegador, terminal y pruebas de conectividad |
| `SW-LAN` | Switch Ethernet de GNS3 | Interconexión de la LAN |
| `NAT-GNS3` | Nodo NAT de GNS3 | Salida WAN simulada |

### 1.3 Limitaciones conocidas

La primera versión del laboratorio utiliza una LAN común para simplificar la implementación. En una red real deben crearse VLAN o subredes independientes para:

- Estudiantes.
- Docentes.
- Servidores.
- Gestión de infraestructura.

El resultado del laboratorio no debe interpretarse como una validación de cobertura Wi-Fi ni como una medición de rendimiento de un AP físico.

---

## 2. Arquitectura del laboratorio

```text
                         +----------------------+
                         | NAT / Internet GNS3  |
                         +----------+-----------+
                                    | ether1 (WAN)
                         +----------v-----------+
                         | R-CORE               |
                         | MikroTik RouterOS 7  |
                         | NAT, FW, DHCP, DNS   |
                         | Hotspot, RADIUS, QoS |
                         +----------+-----------+
                                    | ether2 (LAN)
                              +-----v------+
                              | SW-LAN     |
                              +--+-------+-+
                                 |       |
                         +-------v-+   +-v----------+
                         | AP-01   |   | CLIENTE-01 |
                         | OpenWrt |   | Webterm    |
                         | br-lan  |   +------------+
                         +---------+
```

### Imagen recomendada 1: topología completa

Insertar una captura de la ventana de GNS3 donde se observen:

- Todos los nodos.
- Los nombres `R-CORE`, `AP-01`, `CLIENTE-01`, `SW-LAN` y `NAT-GNS3`.
- Las conexiones entre WAN y LAN.
- El estado iniciado o detenido de cada nodo.

**Ubicación sugerida:** después del diagrama anterior.

**Nombre sugerido:** `docs/img/01-topologia-gns3.png`

**Texto alternativo recomendado:**

> Topología GNS3 con NAT conectado al router MikroTik R-CORE, el switch LAN, el AP OpenWrt y el cliente de pruebas.

---

## 3. Plan de direccionamiento

| Elemento | Dirección o rango | Descripción |
|---|---|---|
| LAN del laboratorio | `192.168.88.0/24` | Segmento común de pruebas |
| Gateway R-CORE | `192.168.88.1` | Puerta de enlace, DNS y Hotspot |
| Gestión AP-01 | `192.168.88.2` | Administración de OpenWrt |
| Servidor interno | `192.168.88.3` | Dirección reservada |
| Pool estudiantes | `192.168.88.4-192.168.88.99` | DHCP principal |
| Pool administrativo | `192.168.88.100-192.168.88.200` | Reservado para VLAN o reservas MAC |
| WAN | DHCP | Dirección recibida desde NAT de GNS3 |

No deben existir dos servidores DHCP en el mismo segmento. El pool administrativo no debe asignarse de manera aleatoria junto con el pool de estudiantes; para ello debe utilizarse una VLAN, una subred diferente, reservas por MAC o una política clara basada en RADIUS.

### Imagen recomendada 2: tabla de direccionamiento

Crear una imagen o captura de una tabla que muestre los equipos, interfaces, IP, máscara y función.

**Ubicación sugerida:** después de la tabla de direccionamiento.

**Nombre sugerido:** `docs/img/02-plan-direccionamiento.png`

---

## 4. Requisitos del laboratorio

### 4.1 Requisitos del host

- Linux con virtualización KVM habilitada.
- GNS3 GUI y GNS3 Server.
- Procesador y memoria suficientes para RouterOS, OpenWrt y el cliente.
- Conexión a Internet para obtener imágenes y paquetes.
- Permisos para ejecutar QEMU/KVM y Docker si se utiliza Webterm.

### 4.2 Imágenes necesarias

| Imagen | Uso | Recomendación |
|---|---|---|
| MikroTik CHR | Router principal | Descargar desde la fuente oficial de MikroTik |
| OpenWrt x86/64 Combined ext4 | AP virtual | Descargar desde `downloads.openwrt.org` |
| Webterm o Linux | Cliente de pruebas | Incluir navegador, `ip`, `dig`, `nslookup` e `iperf3` |

Documentar en el informe:

- Versión exacta de GNS3.
- Versión exacta de RouterOS.
- Versión exacta de OpenWrt.
- Nombre de cada imagen.
- Fecha de descarga.
- Checksum de las imágenes.

No incluir en el repositorio imágenes que tengan restricciones de distribución, certificados, claves privadas, contraseñas ni secretos RADIUS.

---

## 5. Preparación e importación de imágenes

### 5.1 Imagen OpenWrt

1. Acceder a `https://downloads.openwrt.org/`.
2. Seleccionar una versión estable.
3. Entrar en `targets/x86/64/`.
4. Descargar la imagen **Combined ext4** y el archivo de checksum.
5. Verificar el archivo descargado.
6. Descomprimirlo para obtener la imagen `.img`.

```bash
sha256sum openwrt-x86-64-generic-ext4-combined.img.gz
gunzip openwrt-x86-64-generic-ext4-combined.img.gz
```

El nombre puede cambiar de acuerdo con la versión. Debe utilizarse la imagen para `x86/64`, no una imagen destinada a otra arquitectura.

### 5.2 Importar OpenWrt en GNS3

En **Edit > Preferences > QEMU VMs > New**:

1. Crear una nueva VM QEMU.
2. Nombrarla `OpenWrt-AP`.
3. Seleccionar la imagen `.img`.
4. Asignar aproximadamente 256 MB de RAM.
5. Configurar dos adaptadores de red.
6. Utilizar adaptadores VirtIO.
7. Confirmar que el sistema exponga `eth0` y `eth1`.
8. Añadir la VM al proyecto con el nombre `AP-01`.

### 5.3 Importar MikroTik CHR

1. Descargar la imagen CHR compatible con QEMU.
2. Verificar la licencia y el checksum.
3. Crear una VM llamada `R-CORE`.
4. Asignar dos interfaces:
   - `ether1`: WAN hacia NAT-GNS3.
   - `ether2`: LAN hacia SW-LAN.
5. Iniciar la VM y confirmar las interfaces con:

```routeros
/interface print
```

### Imagen recomendada 3: configuración de la VM OpenWrt

Capturar la ventana de configuración de GNS3 donde se vean:

- Nombre de la VM.
- RAM.
- Imagen de disco.
- Número de adaptadores.
- Tipo VirtIO.

**Ubicación sugerida:** al terminar la sección de importación de OpenWrt.

**Nombre sugerido:** `docs/img/03-configuracion-vm-openwrt.png`

### Imagen recomendada 4: configuración de la VM MikroTik

Capturar la configuración de `R-CORE`, mostrando los adaptadores WAN y LAN.

**Nombre sugerido:** `docs/img/04-configuracion-vm-routeros.png`

---

## 6. Creación de la topología

1. Crear el proyecto `colegio-los-robles`.
2. Añadir `NAT-GNS3`, `R-CORE`, `AP-01`, `SW-LAN` y `CLIENTE-01`.
3. Conectar `NAT-GNS3` con `R-CORE/ether1`.
4. Conectar `R-CORE/ether2` con `SW-LAN`.
5. Conectar `AP-01/eth0` con `SW-LAN`.
6. Conectar `CLIENTE-01` con `SW-LAN`.
7. Opcionalmente conectar `AP-01/eth1` a un segundo cliente.
8. Iniciar los nodos en el siguiente orden:
   - SW-LAN.
   - R-CORE.
   - AP-01.
   - CLIENTE-01.

Antes de configurar, confirmar:

```routeros
/interface print
```

```sh
ip link
```

Los nombres de las interfaces pueden variar según la plantilla. No se deben copiar comandos suponiendo que siempre serán `ether1`, `ether2`, `eth0` o `eth1`.

---

## 7. Configuración de R-CORE

### 7.1 Bridge e IP LAN

```routeros
/interface bridge
add name=bridge-lan comment="LAN Colegio Los Robles"

/interface bridge port
add bridge=bridge-lan interface=ether2

/ip address
add address=192.168.88.1/24 interface=bridge-lan comment="Gateway LAN"
```

### 7.2 WAN y DNS

```routeros
/ip dhcp-client
add interface=ether1 disabled=no comment="WAN por DHCP"

/ip dns
set allow-remote-requests=yes
```

En producción, utilizar resolutores autorizados y documentar la política de privacidad.

### 7.3 DHCP

```routeros
/ip pool
add name=pool-estudiantes ranges=192.168.88.4-192.168.88.99

/ip dhcp-server
add name=dhcp-lan interface=bridge-lan address-pool=pool-estudiantes lease-time=8h disabled=no

/ip dhcp-server network
add address=192.168.88.0/24 gateway=192.168.88.1 dns-server=192.168.88.1 comment="Clientes LAN"
```

### 7.4 NAT y firewall

```routeros
/ip firewall nat
add chain=srcnat out-interface=ether1 action=masquerade comment="NAT de salida a Internet"

/ip firewall filter
add chain=input action=accept connection-state=established,related comment="Aceptar conexiones existentes"
add chain=input action=drop connection-state=invalid comment="Descartar conexiones inválidas"
add chain=input action=accept protocol=udp dst-port=67,68 in-interface=bridge-lan comment="Permitir DHCP"
add chain=input action=accept protocol=udp dst-port=53 in-interface=bridge-lan comment="Permitir DNS UDP local"
add chain=input action=accept protocol=tcp dst-port=53 in-interface=bridge-lan comment="Permitir DNS TCP local"
add chain=input action=drop in-interface=ether1 comment="Bloquear acceso entrante desde WAN"
add chain=input action=drop comment="Denegar el resto del tráfico al router"

/ip firewall filter
add chain=forward action=accept connection-state=established,related comment="Forward de conexiones existentes"
add chain=forward action=drop connection-state=invalid
add chain=forward action=accept in-interface=bridge-lan out-interface=ether1 comment="Permitir salida LAN"
add chain=forward action=drop comment="Denegar forward no autorizado"
```

El orden de las reglas es obligatorio. Las reglas de conexiones establecidas deben permanecer antes de los descartes generales.

### 7.5 Redirección DNS

```routeros
/ip firewall nat
add chain=dstnat in-interface=bridge-lan protocol=udp dst-port=53 action=redirect to-ports=53 comment="Forzar DNS UDP local"
add chain=dstnat in-interface=bridge-lan protocol=tcp dst-port=53 action=redirect to-ports=53 comment="Forzar DNS TCP local"
```

### Imagen recomendada 5: interfaces e IP de R-CORE

Capturar en Winbox:

- **Interfaces** con `ether1`, `ether2` y `bridge-lan`.
- **IP > Addresses** con `192.168.88.1/24`.
- **IP > DHCP Client** mostrando WAN `bound`.

**Nombre sugerido:** `docs/img/05-r-core-interfaces.png`

### Imagen recomendada 6: DHCP y NAT

Capturar:

- **IP > DHCP Server > Leases** con el cliente.
- **IP > Firewall > NAT** con la regla masquerade.
- Contadores de paquetes y bytes.

**Nombre sugerido:** `docs/img/06-r-core-dhcp-nat.png`

---

## 8. Configuración del AP OpenWrt

El AP debe operar como puente. No debe competir con R-CORE por DHCP, NAT, DNS o enrutamiento.

### 8.1 Configuración por consola

```sh
passwd

uci set network.lan.proto='static'
uci set network.lan.ipaddr='192.168.88.2'
uci set network.lan.netmask='255.255.255.0'
uci set network.lan.gateway='192.168.88.1'
uci set network.lan.dns='192.168.88.1'
uci set network.lan.device='br-lan'
uci set dhcp.lan.ignore='1'
uci commit network
uci commit dhcp

/etc/init.d/dnsmasq stop
/etc/init.d/dnsmasq disable
/etc/init.d/firewall stop
/etc/init.d/firewall disable
/etc/init.d/network restart
```

### 8.2 Bridge `br-lan`

Comprobar los puertos:

```sh
uci show network | grep -E 'br-lan|eth0|eth1'
bridge link
```

El bridge debe contener el puerto hacia SW-LAN. Si se utiliza el segundo adaptador para un cliente, también debe contener `eth1`.

### 8.3 Configuración mediante LuCI

1. Acceder a `http://192.168.88.2`.
2. Ir a **Network > Interfaces**.
3. Editar la interfaz **LAN**.
4. Seleccionar **Static address**.
5. Configurar IP `192.168.88.2`, máscara `/24`, gateway y DNS `192.168.88.1`.
6. En **Device** o **Physical Settings**, seleccionar `br-lan`.
7. Asociar `eth0` y `eth1` cuando corresponda.
8. Activar **Ignore interface** en el servidor DHCP.
9. Deshabilitar NAT y reglas de firewall que conviertan al AP en router.
10. Aplicar los cambios manteniendo abierta la consola QEMU.

### 8.4 SSID y WPA3

Una VM OpenWrt normalmente no expone una radio inalámbrica. En ese caso, el laboratorio valida solo el bridge Ethernet. Si se utiliza hardware compatible:

- SSID: `LosRobles_WiFi`.
- Modo: Access Point.
- Red: `lan`.
- Cifrado: WPA3-SAE.
- Clave: robusta, única y fuera del repositorio.
- Canal y ancho: conforme al plan de radio.

No se debe declarar que WPA3 fue validado si la VM no tiene una radio compatible.

### Imagen recomendada 7: configuración de red OpenWrt

Capturar LuCI en **Network > Interfaces** mostrando:

- IP de gestión.
- Gateway.
- DNS.
- Bridge `br-lan`.
- DHCP deshabilitado.

**Nombre sugerido:** `docs/img/07-openwrt-red.png`

### Imagen recomendada 8: bridge OpenWrt

Capturar la salida de:

```sh
bridge link
```

o una vista de LuCI donde se observen los puertos del bridge.

**Nombre sugerido:** `docs/img/08-openwrt-bridge.png`

---

## 9. Hotspot, TLS, RADIUS y QoS

### 9.1 Hotspot y certificado

1. Importar el certificado y la clave privada desde **Files**.
2. Ejecutar **IP > Hotspot > Hotspot Setup**.
3. Seleccionar `bridge-lan`.
4. Utilizar un pool exclusivo que no se solape con DHCP.
5. Seleccionar el certificado TLS.
6. Definir un DNS name, por ejemplo `login.losrobles.edu`.

Las claves privadas no deben almacenarse en el repositorio.

### 9.2 RADIUS

Configurar en **Radius**:

- Dirección del servidor RADIUS.
- Servicio `hotspot`.
- Secreto compartido.

Después activar **Use RADIUS** en el perfil del Hotspot. Validar con cuentas de laboratorio y revisar los logs.

### 9.3 Perfiles de velocidad

Crear en **IP > Hotspot > User Profiles**:

| Perfil | Rate Limit | Uso |
|---|---|---|
| `estudiante` | `10M/10M` | Estudiantes |
| `docente` | `30M/30M` | Docentes |

Configurar usuarios de prueba y confirmar en **IP > Hotspot > Active** que el perfil aplicado sea el esperado.

### Imagen recomendada 9: Hotspot y perfiles

Capturar:

- Perfil del servidor Hotspot.
- Certificado seleccionado.
- Perfiles `estudiante` y `docente`.

**Nombre sugerido:** `docs/img/09-hotspot-perfiles.png`

### Imagen recomendada 10: RADIUS y sesiones

Capturar:

- Configuración del servidor RADIUS sin exponer el secreto.
- **IP > Hotspot > Active** con una sesión autenticada.
- **Log** con el resultado de autenticación.

Ocultar nombres de usuario reales, contraseñas, secretos y datos que no sean necesarios.

**Nombre sugerido:** `docs/img/10-radius-sesion.png`

---

## 10. Plan de pruebas

Registrar en una tabla la fecha, prueba, comando, resultado esperado, resultado observado y evidencia.

| ID | Prueba | Resultado esperado |
|---|---|---|
| P01 | Cliente obtiene DHCP | IP entre `192.168.88.4` y `192.168.88.99` |
| P02 | Gateway | Ruta por defecto a `192.168.88.1` |
| P03 | DNS | Resolución mediante R-CORE |
| P04 | NAT | Salida a Internet sin administración WAN |
| P05 | Firewall | Tráfico no autorizado bloqueado |
| P06 | Hotspot | Redirección al portal HTTPS |
| P07 | RADIUS | Usuario válido autenticado |
| P08 | Perfil estudiante | Límite aproximado de 10 Mbps |
| P09 | Perfil docente | Límite aproximado de 30 Mbps |
| P10 | AP | Bridge activo y sin DHCP local |

### 10.1 Pruebas desde el cliente

```bash
ip addr
ip route
nslookup example.com 192.168.88.1
dig @192.168.88.1 example.com
iperf3 -c IP_SERVIDOR_PRUEBAS -t 30
```

### 10.2 Pruebas desde OpenWrt

```sh
ip addr show br-lan
ip route
bridge link
ping -c 4 192.168.88.1
```

### 10.3 Comprobaciones desde RouterOS

```routeros
/ip dhcp-server lease print
/ip firewall nat print stats
/ip firewall filter print stats
/ip hotspot active print
/log print
```

### Imagen recomendada 11: evidencia de pruebas

Presentar una captura del terminal del cliente con:

- Dirección IP recibida.
- Ruta por defecto.
- Resolución DNS.
- Resultado de conectividad.

**Nombre sugerido:** `docs/img/11-pruebas-cliente.png`

### Imagen recomendada 12: QoS

Presentar una captura de `iperf3` o de una herramienta de medición controlada para cada perfil. No mostrar credenciales.

**Nombres sugeridos:**

- `docs/img/12-qos-estudiante.png`
- `docs/img/13-qos-docente.png`

---

## 11. Evidencias visuales y recomendaciones

### 11.1 Estructura sugerida

```text
docs/
└── img/
    ├── 01-topologia-gns3.png
    ├── 02-plan-direccionamiento.png
    ├── 03-configuracion-vm-openwrt.png
    ├── 04-configuracion-vm-routeros.png
    ├── 05-r-core-interfaces.png
    ├── 06-r-core-dhcp-nat.png
    ├── 07-openwrt-red.png
    ├── 08-openwrt-bridge.png
    ├── 09-hotspot-perfiles.png
    ├── 10-radius-sesion.png
    ├── 11-pruebas-cliente.png
    ├── 12-qos-estudiante.png
    └── 13-qos-docente.png
```

### 11.2 Reglas para las capturas

- Usar nombres numerados que sigan el orden del documento.
- Recortar únicamente la ventana relevante.
- Mantener una resolución legible; preferir PNG para pantallas de configuración.
- No capturar claves privadas, contraseñas, secretos RADIUS o tokens.
- Ocultar MAC, nombres de usuario y direcciones públicas si no son necesarios.
- Añadir una breve explicación debajo de cada imagen.
- Mantener el mismo idioma y nombres de equipos en todas las capturas.
- No utilizar imágenes decorativas que no aporten evidencia técnica.
- Comprimir imágenes grandes sin volver ilegible el texto.

### 11.3 Formato para insertar una imagen

Usar rutas relativas desde este documento:

```markdown
![Topología GNS3 del laboratorio](./docs/img/01-topologia-gns3.png)

_Figura 1. Topología implementada en GNS3._
```

### 11.4 Imágenes que no deben incluirse

- Capturas con contraseñas visibles.
- Certificados o claves privadas.
- Archivos `.chr`, `.img` o backups binarios si no está autorizada su distribución.
- Direcciones IP públicas o datos de infraestructura real.
- Capturas repetidas que no demuestren una configuración o resultado.

---

## 12. Solución de problemas

| Problema | Revisión |
|---|---|
| OpenWrt no responde en `192.168.88.2` | Revisar consola, cableado, IP, máscara y bridge. |
| Cliente no recibe DHCP | Confirmar que solo R-CORE tenga DHCP activo y que `eth0` pertenezca a `br-lan`. |
| Cliente recibe `192.168.1.x` | El DHCP de OpenWrt sigue activo. |
| Existe doble NAT | Eliminar masquerade y firewall de OpenWrt. |
| Hay Internet pero no portal | Revisar interfaz del Hotspot, pool, DNS name y certificado. |
| RADIUS rechaza usuarios | Revisar dirección, secreto, hora del sistema y logs. |
| No aparece WPA3 | La VM no tiene radio; verificarlo con hardware físico. |
| QoS no coincide | Revisar perfil activo, tráfico concurrente y servidor de pruebas. |

---

## 13. Lista de comprobación final

- [ ] Las versiones y checksums de las imágenes están documentados.
- [ ] La topología GNS3 coincide con el diagrama.
- [ ] `R-CORE` tiene WAN, LAN, DHCP, NAT y firewall.
- [ ] OpenWrt usa `192.168.88.2/24`.
- [ ] OpenWrt opera como bridge y no entrega DHCP.
- [ ] El cliente obtiene dirección del pool de R-CORE.
- [ ] DNS y salida a Internet funcionan.
- [ ] El acceso administrativo desde WAN está bloqueado.
- [ ] Hotspot y TLS funcionan.
- [ ] RADIUS autentica usuarios de prueba.
- [ ] Los perfiles de 10 y 30 Mbps fueron comprobados.
- [ ] Los logs registran eventos relevantes.
- [ ] Las imágenes del informe no contienen secretos.
- [ ] El export de RouterOS fue sanitizado.
- [ ] Las limitaciones de emulación y WPA3 están documentadas.

---

## 14. Documentos relacionados

- [README.md](./README.md): guía rápida y estructura general del proyecto.
- [GUIA-CONFIGURACION-WINBOX.md](./GUIA-CONFIGURACION-WINBOX.md): configuración gráfica detallada de MikroTik.
- [Paso_8_Configuracion_OpenWrt_AP.md](./Paso_8_Configuracion_OpenWrt_AP.md): instalación y configuración específica de OpenWrt.
