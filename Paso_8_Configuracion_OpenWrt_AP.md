# Configuración del AP virtual OpenWrt

Procedimiento para descargar, importar y configurar OpenWrt como AP virtual en GNS3. En este laboratorio el equipo trabaja como un puente Ethernet de capa 2; MikroTik `R-CORE` conserva el enrutamiento, DHCP, NAT, firewall, Hotspot y RADIUS.

## 1. Parámetros

| Parámetro | Valor |
|---|---|
| Imagen | OpenWrt estable para `x86/64`, Combined ext4 |
| Nombre en GNS3 | `OpenWrt-AP` / `AP-01` |
| RAM | 256 MB |
| Adaptadores | 2, VirtIO |
| Puerto `eth0` | Hacia el switch/R-CORE |
| Puerto `eth1` | Hacia un cliente de prueba |
| Bridge | `br-lan` |
| IP de gestión | `192.168.88.2/24` |
| Gateway y DNS | `192.168.88.1` |
| DHCP, NAT y firewall local | Deshabilitados |
| SSID de referencia | `LosRobles_WiFi` |
| Seguridad física objetivo | WPA3-SAE |

## 2. Descargar y verificar la imagen

1. Acceda a `https://downloads.openwrt.org/`.
2. Seleccione la versión estable y `targets/x86/64/`.
3. Descargue la imagen `combined-ext4` y el checksum publicado.
4. Verifique el archivo antes de importarlo:

```bash
sha256sum openwrt-x86-64-generic-ext4-combined.img.gz
gunzip openwrt-x86-64-generic-ext4-combined.img.gz
```

El nombre puede variar según la versión. No utilice una imagen `ramfs` o de otra arquitectura. Conserve la versión y el checksum en el informe del laboratorio, no necesariamente en el repositorio.

## 3. Crear la VM en GNS3

1. Abra **Edit > Preferences > QEMU VMs > New**.
2. Seleccione **Run the QEMU VM on the GNS3 server**.
3. Nombre la plantilla `OpenWrt-AP`.
4. Seleccione la imagen `.img`.
5. Asigne 256 MB de RAM y dos adaptadores.
6. Seleccione **VirtIO** como tipo de adaptador.
7. Verifique que los nombres de interfaz sean `eth0` y `eth1`.
8. Añada la VM al proyecto como `AP-01`.

En la topología conecte `eth0` al switch LAN. Conecte `eth1` a un cliente solo si se quiere verificar que el AP reenvía tramas entre sus dos puertos.

## 4. Configuración inicial segura

Arranque la VM y acceda por consola. Cambie la contraseña de `root` antes de habilitar LuCI:

```sh
passwd
ip link
ip addr
```

Compruebe los nombres reales de las interfaces. En algunas imágenes pueden aparecer nombres distintos; no copie comandos sin verificar ese punto.

## 5. Configurar el bridge mediante UCI

En imágenes basadas en `netifd`, configure el dispositivo bridge y la interfaz LAN. Si la versión ya tiene `br-lan`, ajuste la sección existente en lugar de crear una duplicada.

```sh
uci set network.lan.proto='static'
uci set network.lan.ipaddr='192.168.88.2'
uci set network.lan.netmask='255.255.255.0'
uci set network.lan.gateway='192.168.88.1'
uci set network.lan.dns='192.168.88.1'
uci set network.lan.device='br-lan'
uci set dhcp.lan.ignore='1'
uci commit network
uci commit dhcp
```

En OpenWrt con configuración DSA, confirme que el dispositivo tenga los puertos físicos:

```sh
uci show network | grep -E 'br-lan|eth0|eth1'
bridge link
```

Si `br-lan` no incluye ambos puertos, cree o ajuste el dispositivo desde LuCI. Como alternativa, con la VM todavía accesible por consola, puede crear una sección nueva:

```sh
uci -q delete network.br_lan
uci set network.br_lan='device'
uci set network.br_lan.name='br-lan'
uci set network.br_lan.type='bridge'
uci add_list network.br_lan.ports='eth0'
uci add_list network.br_lan.ports='eth1'
uci set network.lan.device='br-lan'
uci commit network
```

Revise con `uci show network` que no existan dos secciones que intenten administrar `br-lan`. Las interfaces de la VM pueden variar entre plantillas; sustitúyalas por los nombres mostrados por `ip link`.

## 6. Desactivar servicios que pertenecen a R-CORE

OpenWrt no debe entregar una segunda dirección, hacer NAT ni filtrar el tráfico de la LAN:

```sh
/etc/init.d/dnsmasq stop
/etc/init.d/dnsmasq disable
/etc/init.d/firewall stop
/etc/init.d/firewall disable
/etc/init.d/network restart
```

Después del reinicio, vuelva a conectarse a `http://192.168.88.2` o por SSH. Si se pierde la conexión, use la consola QEMU para corregir el bridge y la IP.

## 7. Configuración equivalente mediante LuCI

1. Conecte un cliente al mismo segmento y abra `http://192.168.88.2`.
2. En **Network > Interfaces**, edite **LAN**.
3. Seleccione **Static address**.
4. Configure:
   - IPv4: `192.168.88.2`
   - Netmask: `255.255.255.0`
   - Gateway: `192.168.88.1`
   - DNS: `192.168.88.1`
5. En **Device** o **Physical Settings**, seleccione el bridge `br-lan` y asocie `eth0` y `eth1`.
6. En **DHCP Server**, marque **Ignore interface**.
7. En **Network > Firewall**, elimine o desactive el masquerading y las zonas de router que no correspondan al bridge.
8. Pulse **Save & Apply** y confirme la conectividad.

No desactive el acceso administrativo hasta comprobar que la nueva IP responde. Mantenga abierta la consola QEMU durante el cambio.

## 8. SSID y WPA3

La VM QEMU normalmente no tiene una radio inalámbrica. En ese caso, OpenWrt solo prueba el puente Ethernet y no debe presentarse como una validación real de WPA3.

Si se dispone de hardware o una radio virtual compatible, configure el SSID desde **Network > Wireless**:

- SSID: `LosRobles_WiFi`
- Modo: `Access Point`
- Red: `lan`
- Canal y ancho: según el plan de radio; preferir 20 MHz en 2.4 GHz cuando haya interferencia.
- Cifrado: `WPA3-SAE` (`sae`) si el hardware y los clientes lo soportan.
- Clave: robusta, exclusiva y fuera del repositorio.

Ejemplo UCI para una imagen que expone `radio0`:

```sh
uci set wireless.radio0.disabled='0'
uci set wireless.radio0.channel='auto'
uci set wireless.default_radio0.mode='ap'
uci set wireless.default_radio0.ssid='LosRobles_WiFi'
uci set wireless.default_radio0.network='lan'
uci set wireless.default_radio0.encryption='sae'
uci set wireless.default_radio0.key='CAMBIAR_CLAVE'
uci commit wireless
/etc/init.d/network reload
```

No ejecute este bloque si `radio0` no existe. En GNS3, documente WPA3 como requisito de la fase física.

## 9. Verificación

Ejecute en OpenWrt:

```sh
ip addr show br-lan
ip route
bridge link
ping -c 4 192.168.88.1
logread | tail -n 30
```

Resultados esperados:

- `br-lan` tiene `192.168.88.2/24`.
- La ruta por defecto apunta a `192.168.88.1`.
- `eth0` y, si se utiliza, `eth1` pertenecen al bridge.
- El gateway responde desde el AP.
- No hay un servidor DHCP local activo.

Desde el cliente:

```bash
ip addr
ip route
nslookup example.com 192.168.88.1
```

El cliente debe recibir una IP del pool de R-CORE. En MikroTik compruebe `/ip dhcp-server lease print` y que la IP `192.168.88.2` quede reservada para el AP.

## 10. Integración con Hotspot

La dirección de gestión del AP puede quedar fuera del portal cautivo:

1. En **IP > Hotspot > IP Bindings**, cree una entrada para `192.168.88.2`.
2. Utilice la MAC real del AP y el tipo `bypassed` únicamente si la política de administración lo requiere.
3. En **IP > ARP**, agregue una entrada estática solo después de verificar la MAC.

No copie MAC de ejemplo. En una red grande, no active `arp=reply-only` sin reservas y entradas ARP para todos los equipos autorizados.

## 11. Problemas frecuentes

| Problema | Solución |
|---|---|
| No responde `192.168.88.2` | Revisar consola, `ip addr`, cableado y que la IP no esté siendo usada por otro nodo. |
| El cliente obtiene una IP `192.168.1.x` | Quedó activo el DHCP de OpenWrt; revisar `dhcp.lan.ignore` y `dnsmasq`. |
| El cliente no obtiene IP | Revisar que `eth0` esté en `br-lan`, que R-CORE tenga DHCP activo y que el switch esté encendido. |
| El cliente tiene Internet pero no portal | El Hotspot está en la interfaz incorrecta o el pool se solapa; revisar R-CORE. |
| No aparece WPA3 | La VM no tiene radio compatible; validar únicamente bridge en GNS3 y WPA3 en hardware real. |

## 12. Criterios de aceptación del AP

- [ ] Imagen x86/64 descargada y checksum verificado.
- [ ] VM QEMU importada con dos adaptadores VirtIO.
- [ ] IP de gestión `192.168.88.2/24` configurada.
- [ ] `br-lan` contiene el puerto hacia R-CORE.
- [ ] DHCP, NAT y firewall local están deshabilitados.
- [ ] El cliente recibe DHCP desde R-CORE.
- [ ] El AP tiene conectividad hacia `192.168.88.1`.
- [ ] La clave inalámbrica no se publicó.
- [ ] WPA3 se validó en hardware si se declara como resultado del proyecto.
