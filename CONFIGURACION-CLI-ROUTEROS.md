# Configuración por comandos de RouterOS

Este documento contiene la alternativa por CLI para el laboratorio. La implementación realizada con el grupo se documenta en [GUIA-CONFIGURACION-WINBOX.md](./GUIA-CONFIGURACION-WINBOX.md); no es necesario ejecutar ambos métodos sobre el mismo router porque se crearían configuraciones duplicadas.

> Ejecute los bloques por etapas, confirme los nombres reales de las interfaces y sustituya todos los valores marcados como ejemplo. No pegue un bloque completo sin revisar la configuración existente.

## 1. Variables del escenario

| Elemento | Valor |
|---|---|
| WAN | `ether1` |
| LAN | `ether2` |
| Bridge | `bridge-lan` |
| Gateway | `192.168.88.1/24` |
| Pool clientes | `192.168.88.4-192.168.88.99` |
| AP | `192.168.88.2/24` |

Antes de comenzar:

```routeros
/interface print
/ip address print
/export hide-sensitive
```

## 2. Identidad, bridge y WAN

```routeros
/system identity set name=R-CORE

/interface bridge
add name=bridge-lan comment="LAN Colegio Los Robles"

/interface bridge port
add bridge=bridge-lan interface=ether2

/ip address
add address=192.168.88.1/24 interface=bridge-lan comment="Gateway LAN"

/ip dhcp-client
add interface=ether1 disabled=no comment="WAN por DHCP"
```

No agregue `ether1` al bridge. Compruebe que el DHCP client esté en estado `bound`.

## 3. DNS y DHCP

```routeros
/ip dns
set servers=1.1.1.1,8.8.8.8 allow-remote-requests=yes

/ip pool
add name=pool-estudiantes ranges=192.168.88.4-192.168.88.99

/ip dhcp-server
add name=dhcp-lan interface=bridge-lan address-pool=pool-estudiantes lease-time=8h disabled=no

/ip dhcp-server network
add address=192.168.88.0/24 gateway=192.168.88.1 dns-server=192.168.88.1 comment="Clientes LAN"
```

## 4. NAT, firewall y DNS forzado

```routeros
/ip firewall nat
add chain=srcnat out-interface=ether1 action=masquerade comment="NAT de salida a Internet"
add chain=dstnat in-interface=bridge-lan protocol=udp dst-port=53 action=redirect to-ports=53 comment="Forzar DNS UDP local"
add chain=dstnat in-interface=bridge-lan protocol=tcp dst-port=53 action=redirect to-ports=53 comment="Forzar DNS TCP local"

/ip firewall filter
add chain=input action=accept connection-state=established,related comment="Aceptar conexiones existentes"
add chain=input action=drop connection-state=invalid comment="Descartar conexiones inválidas"
add chain=input action=accept protocol=udp dst-port=67,68 in-interface=bridge-lan comment="Permitir DHCP"
add chain=input action=accept protocol=udp dst-port=53 in-interface=bridge-lan comment="Permitir DNS UDP local"
add chain=input action=accept protocol=tcp dst-port=53 in-interface=bridge-lan comment="Permitir DNS TCP local"
add chain=input action=drop in-interface=ether1 comment="Bloquear acceso entrante desde WAN"
add chain=input action=drop comment="Denegar el resto del tráfico al router"
add chain=forward action=accept connection-state=established,related comment="Forward de conexiones existentes"
add chain=forward action=drop connection-state=invalid
add chain=forward action=accept in-interface=bridge-lan out-interface=ether1 comment="Permitir salida LAN"
add chain=forward action=drop comment="Denegar forward no autorizado"
```

Revise el orden de las reglas y sus contadores:

```routeros
/ip firewall filter print stats
/ip firewall nat print stats
```

## 5. ARP y gestión del AP

Use la MAC observada en el laboratorio, nunca la de un ejemplo:

```routeros
/ip arp
add address=192.168.88.2 mac-address=AA:BB:CC:DD:EE:FF interface=bridge-lan comment="AP-01"
```

Si el AP debe quedar fuera del portal:

```routeros
/ip hotspot ip-binding
add address=192.168.88.2 type=bypassed comment="Gestión AP-01"
```

Ejecute esta parte solo después de configurar el Hotspot y confirmar la política de administración.

## 6. Crear certificados para el Hotspot

La versión gráfica equivalente está en la sección 11 de [GUIA-CONFIGURACION-WINBOX.md](./GUIA-CONFIGURACION-WINBOX.md). Para un certificado autofirmado de laboratorio:

```routeros
/certificate
add name=CA-LosRobles common-name=CA-LosRobles days-valid=365 key-usage=key-cert-sign,crl-sign
sign CA-LosRobles

add name=Hotspot-LosRobles common-name=login.losrobles.edu subject-alt-name=DNS:login.losrobles.edu,IP:192.168.88.1 days-valid=365 key-usage=digital-signature,key-encipherment,tls-server
sign Hotspot-LosRobles ca=CA-LosRobles

/certificate
set [find name=CA-LosRobles] trusted=yes
set [find name=Hotspot-LosRobles] trusted=yes
```

Compruebe el resultado:

```routeros
/certificate print detail
```

Un certificado autofirmado produce advertencia en los clientes hasta que se instale la CA pública `CA-LosRobles` como autoridad de confianza. No exporte la clave privada del servidor:

```routeros
/certificate export-certificate CA-LosRobles export-passphrase=""
```

Descargue únicamente el `.crt` de la CA e instálelo en el cliente del laboratorio.

## 7. Hotspot, RADIUS y perfiles

Ejecute el asistente y confirme cada valor antes de continuar:

```routeros
/ip hotspot setup
```

Seleccione `bridge-lan`, una dirección `192.168.88.1/24`, un pool que no se solape con DHCP y el certificado `Hotspot-LosRobles`.

```routeros
/radius
add address=127.0.0.1 service=hotspot secret="CAMBIAR_SECRET_RADIUS"

/ip hotspot profile
set [find name="default"] ssl-certificate=Hotspot-LosRobles use-radius=yes login-by=https,http-chap

/ip hotspot user profile
add name=estudiante rate-limit=10M/10M shared-users=1
add name=docente rate-limit=30M/30M shared-users=1
```

La dirección `127.0.0.1` solo es correcta si User Manager está en el mismo router. Cambie la dirección si RADIUS está en otro servidor.

## 8. Registros y respaldo

```routeros
/system logging
add topics=firewall action=memory
add topics=hotspot,account action=memory
add topics=radius action=memory

/export hide-sensitive file=export-sanitizado
/system backup save name=backup-laboratorio
/log print
```

Los respaldos y exports deben mantenerse fuera del repositorio si contienen información sensible.

## 9. Verificación

```routeros
/ip dhcp-client print
/ip dhcp-server lease print
/ip route print
/ip hotspot active print
/ip firewall filter print stats
/ip firewall nat print stats
/log print
```

Desde el cliente:

```bash
ip addr
ip route
nslookup example.com 192.168.88.1
dig @8.8.8.8 example.com
iperf3 -c IP_SERVIDOR_PRUEBAS -t 30
```

Valide que el cliente reciba DHCP, que NAT funcione, que el portal use HTTPS y que los perfiles se aproximen a 10 Mbps y 30 Mbps, respectivamente.
