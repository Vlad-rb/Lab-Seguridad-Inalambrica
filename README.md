# Red inalámbrica segura en GNS3

Guía de laboratorio para implementar una red inalámbrica segura para el Colegio Los Robles utilizando GNS3, MikroTik RouterOS y un AP virtual OpenWrt.

> **Importante:** GNS3 valida el direccionamiento, el enrutamiento, el firewall, el DHCP, el portal cautivo, RADIUS, QoS y el funcionamiento del AP como puente. Una VM OpenWrt con interfaces Ethernet no emula por sí sola la cobertura, interferencia, roaming ni la negociación WPA3 de una radio física. Esas características deben validarse con hardware compatible.

Para la documentación formal completa, incluyendo el procedimiento, las pruebas y las recomendaciones para insertar evidencias visuales, consulte [DOCUMENTACION-LABORATORIO.md](./DOCUMENTACION-LABORATORIO.md). La implementación principal del laboratorio se realizó con [GUIA-CONFIGURACION-WINBOX.md](./GUIA-CONFIGURACION-WINBOX.md); la variante por comandos está en [CONFIGURACION-CLI-ROUTEROS.md](./CONFIGURACION-CLI-ROUTEROS.md).

## Índice

1. [Objetivo y alcance](#1-objetivo-y-alcance)
2. [Arquitectura](#2-arquitectura)
3. [Direccionamiento](#3-direccionamiento)
4. [Requisitos](#4-requisitos)
5. [Preparación de imágenes](#5-preparación-de-imágenes)
6. [Creación de la topología](#6-creación-de-la-topología)
7. [Orden de implementación](#7-orden-de-implementación)
8. [Configuración del AP OpenWrt](#8-configuración-del-ap-openwrt)
9. [Validación](#9-validación)
10. [Solución de problemas](#10-solución-de-problemas)
11. [Seguridad y entregables](#11-seguridad-y-entregables)

## 1. Objetivo y alcance

El laboratorio representa una red para estudiantes, docentes y personal administrativo. El router centraliza:

- WAN, NAT y gateway de la LAN.
- DHCP y DNS para los clientes.
- Firewall de entrada y reenvío.
- Hotspot con portal cautivo HTTPS.
- Autenticación RADIUS/User Manager.
- Perfiles de velocidad: 10 Mbps para estudiantes y 30 Mbps para docentes.
- Registro de eventos de firewall, Hotspot y RADIUS.

La primera versión utiliza una LAN común para simplificar la emulación. En producción se deben separar como mínimo las VLAN de estudiantes, docentes, servidores y gestión.

## 2. Arquitectura

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
                                    | ether2
                              +-----v------+
                              | Switch LAN |
                              +--+-------+-+
                                 |       |
                         +-------v-+   +-v----------+
                         | AP-01   |   | CLIENTE-01 |
                         | OpenWrt |   | Webterm    |
                         | br-lan  |   +------------+
                         +---------+
```

Conecte `eth0` de OpenWrt hacia el switch y `eth1` hacia un segundo cliente si se desea comprobar el puente L2. En una topología con un único enlace al switch, `eth1` puede quedar reservado para pruebas.

## 3. Direccionamiento

| Elemento | Dirección o rango | Uso |
|---|---|---|
| LAN | `192.168.88.0/24` | Red del laboratorio |
| R-CORE | `192.168.88.1` | Gateway, DNS y Hotspot |
| AP-01 | `192.168.88.2` | Gestión de OpenWrt |
| SRV-RADIUS | `192.168.88.3` | Ubuntu Desktop y FreeRADIUS externo |
| Pool estudiantes | `192.168.88.4-192.168.88.99` | DHCP principal |
| Pool administrativo | `192.168.88.100-192.168.88.200` | Reservado para una futura VLAN o reservas MAC |
| WAN | DHCP de NAT GNS3 | Salida a Internet |

No asigne el pool administrativo en la misma red de manera aleatoria. Para diferenciar perfiles automáticamente utilice VLAN/subred separada, reservas DHCP por MAC o autenticación Hotspot/RADIUS.

## 4. Requisitos

### Host

- Linux con virtualización KVM habilitada.
- GNS3 GUI y GNS3 Server.
- CPU y memoria suficientes para dos VMs y un cliente Webterm.
- Conectividad a Internet para descargar imágenes y paquetes.

### Imágenes y credenciales

- MikroTik CHR RouterOS 7.x, descargado desde la fuente oficial.
- OpenWrt x86/64 para QEMU/KVM.
- Ubuntu Desktop para el servidor externo FreeRADIUS.
- Webterm Docker o una VM Linux con navegador y herramientas `ip`, `dig`, `nslookup` e `iperf3`.
- Certificado TLS para el Hotspot, únicamente en el entorno local.
- Datos de una cuenta administrativa inicial y secreto RADIUS de laboratorio.

Nunca publique imágenes con licencias restringidas, certificados, claves privadas, contraseñas, exports sin sanitizar ni secretos RADIUS.

## 5. Preparación de imágenes

### 5.1 OpenWrt

1. Descargue la imagen x86/64 **Combined ext4** desde `https://downloads.openwrt.org/`.
2. Elija la versión estable y la ruta `targets/x86/64/`.
3. Descargue el archivo `openwrt-x86-64-generic-ext4-combined.img.gz` o el nombre equivalente de la versión elegida.
4. Verifique el checksum publicado por OpenWrt.
5. Descomprima el archivo para obtener un `.img`.

Ejemplo en Linux:

```bash
sha256sum openwrt-x86-64-generic-ext4-combined.img.gz
gunzip openwrt-x86-64-generic-ext4-combined.img.gz
```

El nombre exacto puede cambiar entre versiones; utilice siempre el archivo correspondiente a `x86/64` y documente la versión usada.

### 5.2 MikroTik CHR

1. Descargue la imagen CHR compatible con QEMU desde el sitio oficial de MikroTik.
2. Verifique la licencia y el checksum.
3. Importe la imagen en GNS3 como VM QEMU.
4. Asigne al menos dos adaptadores: WAN y LAN.

### 5.3 Plantillas en GNS3

En **Edit > Preferences > QEMU VMs > New**:

- `OpenWrt-AP`: 256 MB de RAM, dos adaptadores VirtIO y la imagen `.img`.
- `R-CORE`: memoria según la imagen CHR, dos adaptadores VirtIO.
- Cliente: Webterm o VM con navegador.

En cada VM confirme el orden de interfaces antes de cablear. El nombre `eth0` debe documentarse como el puerto conectado a R-CORE y `eth1` como el puerto de prueba.

## 6. Creación de la topología

1. Cree un proyecto llamado `colegio-los-robles`.
2. Añada un nodo NAT de GNS3, `R-CORE`, `AP-01`, un switch Ethernet y `CLIENTE-01`.
3. Conecte:
   - NAT GNS3 a `R-CORE/ether1`.
   - `R-CORE/ether2` al switch.
   - `AP-01/eth0` al switch.
   - `CLIENTE-01` al switch.
4. Inicie primero el switch, después `R-CORE`, OpenWrt y el cliente.
5. Abra las consolas y confirme las interfaces con `/interface print` en RouterOS y `ip link` en OpenWrt.

## 7. Orden de implementación

Realice los cambios en este orden para evitar perder acceso:

1. Cambie las credenciales iniciales y haga un respaldo.
2. Configure el bridge LAN y la IP `192.168.88.1/24` en R-CORE.
3. Configure WAN por DHCP, DNS, DHCP, NAT y firewall.
4. Configure OpenWrt como bridge con la IP `192.168.88.2`.
5. Configure Ubuntu Desktop como servidor FreeRADIUS externo con la IP `192.168.88.3`.
6. Compruebe que el cliente recibe DHCP y tiene salida a Internet.
7. Configure Hotspot, certificado TLS y RADIUS.
8. Cree los perfiles `estudiante` y `docente`.
9. Active los registros y ejecute la matriz de pruebas.

La configuración gráfica completa del router está en [GUIA-CONFIGURACION-WINBOX.md](./GUIA-CONFIGURACION-WINBOX.md). La configuración específica del AP está en [Paso_8_Configuracion_OpenWrt_AP.md](./Paso_8_Configuracion_OpenWrt_AP.md).

## 8. Configuración del AP OpenWrt

OpenWrt funciona en modo **bridge transparente**: no entrega DHCP, no hace NAT y no enruta a Internet. Todo el tráfico pasa hacia R-CORE.

Desde la consola de OpenWrt:

```sh
# Sustituir eth0/eth1 si la VM muestra otros nombres.
uci set network.lan.proto='static'
uci set network.lan.ipaddr='192.168.88.2'
uci set network.lan.netmask='255.255.255.0'
uci set network.lan.gateway='192.168.88.1'
uci set network.lan.dns='192.168.88.1'
uci set dhcp.lan.ignore='1'
uci commit network
uci commit dhcp
/etc/init.d/dnsmasq disable
/etc/init.d/dnsmasq stop
/etc/init.d/firewall disable
/etc/init.d/firewall stop
```

En versiones modernas de OpenWrt, compruebe en LuCI que el dispositivo `br-lan` tenga como puertos `eth0` y `eth1`. Si no existe, créelo en **Network > Interfaces > Devices** como bridge y asócielo a la interfaz LAN. No reinicie la red hasta confirmar que el puerto de administración está conectado.

La configuración completa, alternativa por LuCI, SSID y comprobaciones se encuentra en [Paso_8_Configuracion_OpenWrt_AP.md](./Paso_8_Configuracion_OpenWrt_AP.md).

## 9. Validación

### Cliente

```bash
ip addr
ip route
nslookup example.com 192.168.88.1
dig @192.168.88.1 example.com
```

Debe recibir una dirección entre `192.168.88.4` y `192.168.88.99`, gateway `192.168.88.1` y resolución DNS.

### OpenWrt

```sh
ip addr show br-lan
ip route
bridge link
ping -c 4 192.168.88.1
```

La IP de gestión debe ser `192.168.88.2/24`. `bridge link` debe mostrar los puertos asociados a `br-lan`.

### RouterOS

```routeros
/interface print
/ip dhcp-server lease print
/ip firewall nat print stats
/ip firewall filter print stats
/ip hotspot active print
/log print
```

Compruebe que WAN esté activa, que el cliente tenga una concesión, que los contadores de NAT/firewall aumenten y que las sesiones Hotspot se registren.

### Portal, RADIUS y QoS

1. Navegue desde el cliente a un sitio HTTP de prueba.
2. Confirme la redirección al portal HTTPS.
3. Pruebe una cuenta de estudiante y otra de docente.
4. Ejecute `iperf3` contra un servidor controlado:

```bash
iperf3 -c IP_SERVIDOR_PRUEBAS -t 30
```

El perfil estudiante no debe superar aproximadamente 10 Mbps y el docente aproximadamente 30 Mbps. Las mediciones del laboratorio no representan una garantía de velocidad física.

## 10. Solución de problemas

| Síntoma | Comprobaciones |
|---|---|
| OpenWrt no responde en `192.168.88.2` | Revisar cableado, `ip addr`, bridge, máscara y gateway. Conectar por consola antes de reiniciar la red. |
| Cliente no recibe DHCP | Confirmar que solo R-CORE tenga DHCP activo y que `eth0`/`eth1` estén dentro de `br-lan`. |
| Hay doble NAT | Eliminar masquerade, DHCP y firewall de OpenWrt; el único NAT debe estar en R-CORE. |
| Hay Internet pero no portal | Revisar Hotspot, DNS name, certificado, pool exclusivo y estado de la interfaz `bridge-lan`. |
| RADIUS rechaza usuarios | Revisar dirección, secreto, servicio Hotspot, hora del sistema y logs de ambos extremos. |
| QoS no coincide | Confirmar el perfil activo y repetir la prueba contra un servidor local, sin tráfico concurrente. |
| WPA3 no aparece | La VM probablemente no tiene radio. Documentar el bridge y validar WPA3 con hardware real. |

## 11. Seguridad y entregables

Antes de entregar el laboratorio:

- [ ] Cambiar contraseñas iniciales y usar secretos únicos.
- [ ] No guardar claves TLS, secretos RADIUS ni contraseñas en Git.
- [ ] Exportar RouterOS solo después de sanitizarlo.
- [ ] Verificar que la administración no sea accesible desde WAN.
- [ ] Mantener respaldos fuera del repositorio.
- [ ] Registrar versión de GNS3, RouterOS, OpenWrt, imágenes y fecha de pruebas.
- [ ] Adjuntar evidencias de DHCP, NAT, portal, RADIUS, QoS y logs.

El resultado esperado es una emulación reproducible de la lógica de red. Para producción todavía deben planificarse VLAN, cobertura, canales, capacidad, certificados confiables, monitoreo, redundancia y validación WPA3 con AP físico.
