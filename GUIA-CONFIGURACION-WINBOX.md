# Guía de configuración MikroTik con Winbox

Guía gráfica para configurar el router MikroTik del proyecto **Colegio Los Robles** mediante **Winbox** y RouterOS 7.x. El procedimiento del AP OpenWrt está separado en [Paso_8_Configuracion_OpenWrt_AP.md](./Paso_8_Configuracion_OpenWrt_AP.md).

## 1. Alcance y precauciones

Esta guía cubre:

- Router principal `R-CORE`.
- AP `AP-01` en modo bridge cuando se utilice un segundo MikroTik.
- DHCP, NAT, firewall, DNS, Hotspot, RADIUS, QoS y registros.

Antes de comenzar:

1. Ingresar al equipo mediante Winbox usando MAC o IP.
2. Confirmar las interfaces reales en **Interfaces**.
3. Crear un respaldo en **Files** o desde **System > Backup**.
4. Sustituir todas las direcciones MAC, certificados, nombres y secretos de ejemplo.
5. Aplicar los cambios primero en el laboratorio GNS3.

> Los nombres de menú pueden variar ligeramente según la versión de Winbox y RouterOS. Esta guía está orientada a RouterOS 7.x.

## 2. Datos del escenario

| Elemento | Valor |
|---|---|
| WAN del router | `ether1` |
| LAN del router | `ether2` |
| Bridge LAN | `bridge-lan` |
| Gateway | `192.168.88.1/24` |
| Pool estudiantes | `192.168.88.4-192.168.88.99` |
| Pool administrativo | `192.168.88.100-192.168.88.200` |
| Gestión AP | `192.168.88.2/24` |
| Servidor RADIUS Ubuntu | `192.168.88.3/24` |
| Perfil estudiante | `10M/10M` |
| Perfil docente | `30M/30M` |

## 3. Acceso inicial mediante Winbox

1. Abrir **Winbox**.
2. En la pestaña **Neighbors**, localizar el equipo por su MAC.
3. Seleccionar el dispositivo y pulsar **Connect**.
4. Autenticarse con la cuenta administrativa inicial.
5. Cambiar inmediatamente la contraseña desde **System > Users**.
6. Confirmar el modelo y la versión en **System > Resources**.

Para administrar por IP, utilizar la dirección configurada en la interfaz de gestión. No exponer Winbox a Internet.

## 4. Configuración del router principal `R-CORE`

### 4.1 Renombrar el equipo

1. Ir a **System > Identity**.
2. En **Identity**, escribir `R-CORE`.
3. Pulsar **Apply** y **OK**.

### 4.2 Crear el bridge LAN

1. Ir a **Bridge > Bridge**.
2. Pulsar **+**.
3. En **Name**, escribir `bridge-lan`.
4. En **Comment**, escribir `LAN Colegio Los Robles`.
5. Pulsar **Apply > OK**.

Agregar la interfaz LAN:

1. Ir a **Bridge > Ports**.
2. Pulsar **+**.
3. En **Interface**, seleccionar `ether2`.
4. En **Bridge**, seleccionar `bridge-lan`.
5. Pulsar **Apply > OK**.

No agregar `ether1` al bridge si se utilizará como WAN.

### 4.3 Configurar la dirección LAN

1. Ir a **IP > Addresses**.
2. Pulsar **+**.
3. En **Address**, escribir `192.168.88.1/24`.
4. En **Interface**, seleccionar `bridge-lan`.
5. En **Comment**, escribir `Gateway LAN`.
6. Pulsar **Apply > OK**.

### 4.4 Configurar la WAN

Si la WAN de GNS3 entrega dirección por DHCP:

1. Ir a **IP > DHCP Client**.
2. Pulsar **+**.
3. En **Interface**, seleccionar `ether1`.
4. Activar **Add Default Route**.
5. Activar **Use Peer DNS** solo si se desea utilizar el DNS entregado por la WAN.
6. Pulsar **Apply > OK**.

Verificar que el estado sea **bound**.

## 5. Configuración del servidor DNS

1. Ir a **IP > DNS**.
2. En **Servers**, introducir los resolutores autorizados, por ejemplo:
   - `1.1.1.1`
   - `8.8.8.8`
3. Activar **Allow Remote Requests** para que los clientes consulten al router.
4. Pulsar **Apply > OK**.

En producción, reemplazar los resolutores públicos por el servidor DNS institucional o un servicio con filtrado aprobado.

## 6. Configuración de DHCP

### 6.1 Crear los pools

1. Ir a **IP > Pool**.
2. Pulsar **+**.
3. Crear:
   - **Name:** `pool-estudiantes`
   - **Addresses:** `192.168.88.4-192.168.88.99`
4. Pulsar **Apply > OK**.
5. Pulsar nuevamente **+** y crear:
   - **Name:** `pool-administrativo`
   - **Addresses:** `192.168.88.100-192.168.88.200`
6. Pulsar **Apply > OK**.

Las IP `192.168.88.2` y `192.168.88.3` quedan reservadas para infraestructura.

### 6.2 Crear el servidor DHCP

1. Ir a **IP > DHCP Server**.
2. Pulsar **DHCP Setup**.
3. Seleccionar `bridge-lan`.
4. En **DHCP Address Space**, confirmar `192.168.88.0/24`.
5. En **Gateway for DHCP Network**, confirmar `192.168.88.1`.
6. En **Addresses to Give Out**, seleccionar `192.168.88.4-192.168.88.99`.
7. En **DNS Servers**, escribir `192.168.88.1`.
8. Definir un tiempo de concesión de `8h`.
9. Finalizar con **Next** hasta **Close**.

El pool administrativo no debe asignarse arbitrariamente dentro de la misma subred. Para diferenciarlo automáticamente se recomienda una VLAN/subred propia o reservas DHCP por MAC.

## 7. NAT de salida

1. Ir a **IP > Firewall > NAT**.
2. Pulsar **+**.
3. En la pestaña **General**:
   - **Chain:** `srcnat`
   - **Out. Interface:** `ether1`
4. En la pestaña **Action**:
   - **Action:** `masquerade`
5. En **Comment**, escribir `NAT de salida a Internet`.
6. Pulsar **Apply > OK**.

## 8. Firewall mediante Winbox

Ir a **IP > Firewall > Filter Rules**. Crear las reglas con **+** y respetar este orden.

### 8.1 Reglas de entrada al router

#### Regla 1 — Conexiones existentes

- **General > Chain:** `input`
- **Connection State:** marcar `established` y `related`
- **Action:** `accept`
- Comentario: `Aceptar conexiones existentes`

#### Regla 2 — Conexiones inválidas

- **Chain:** `input`
- **Connection State:** `invalid`
- **Action:** `drop`

#### Regla 3 — DHCP

- **Chain:** `input`
- **Protocol:** `udp`
- **Dst. Port:** `67,68`
- **In. Interface:** `bridge-lan`
- **Action:** `accept`

#### Regla 4 — DNS UDP

- **Chain:** `input`
- **Protocol:** `udp`
- **Dst. Port:** `53`
- **In. Interface:** `bridge-lan`
- **Action:** `accept`

#### Regla 5 — DNS TCP

- **Chain:** `input`
- **Protocol:** `tcp`
- **Dst. Port:** `53`
- **In. Interface:** `bridge-lan`
- **Action:** `accept`

#### Regla 6 — Bloquear ping al gateway

- **Chain:** `input`
- **Protocol:** `icmp`
- **Dst. Address:** `192.168.88.1`
- **Action:** `drop`
- Comentario: `Bloquear ICMP al gateway`

#### Regla 7 — Bloquear administración desde WAN

- **Chain:** `input`
- **In. Interface:** `ether1`
- **Action:** `drop`
- Comentario: `Bloquear acceso entrante desde WAN`

#### Regla 8 — Denegar el resto

- **Chain:** `input`
- **Action:** `drop`
- Comentario: `Denegar el resto del tráfico al router`

### 8.2 Reglas de reenvío

En la misma ventana crear:

1. `forward`, estados `established,related`, acción `accept`.
2. `forward`, estado `invalid`, acción `drop`.
3. `forward`, **In. Interface** `bridge-lan`, **Out. Interface** `ether1`, acción `accept`.
4. `forward`, acción `drop`.

Mover reglas con las flechas de Winbox si el orden no es correcto. Verificar contadores en las columnas **Packets** y **Bytes**.

## 9. Redirección de DNS

Ir a **IP > Firewall > NAT** y crear dos reglas.

### DNS UDP

- **Chain:** `dstnat`
- **Protocol:** `udp`
- **Dst. Port:** `53`
- **In. Interface:** `bridge-lan`
- **Action:** `redirect`
- **To Ports:** `53`
- Comentario: `Forzar DNS UDP local`

### DNS TCP

- **Chain:** `dstnat`
- **Protocol:** `tcp`
- **Dst. Port:** `53`
- **In. Interface:** `bridge-lan`
- **Action:** `redirect`
- **To Ports:** `53`
- Comentario: `Forzar DNS TCP local`

El DNS estándar usa los puertos 53 TCP/UDP. Un puerto aleatorio no debe utilizarse como prueba de redirección DNS.

## 10. Entradas ARP estáticas

1. Ir a **IP > ARP**.
2. Pulsar **+**.
3. Para el AP:
   - **Address:** `192.168.88.2`
   - **MAC Address:** MAC real del AP
   - **Interface:** `bridge-lan`
4. Para el servidor:
   - **Address:** `192.168.88.3`
   - **MAC Address:** MAC real del servidor
   - **Interface:** `bridge-lan`
5. Pulsar **Apply > OK** en cada entrada.

No utilizar las MAC de esta guía. La opción **ARP = reply-only** solo debe activarse cuando todos los equipos autorizados tengan reservas DHCP y entradas ARP.

## 11. Hotspot y portal cautivo

### 11.1 Crear un certificado autofirmado desde Winbox

Para el laboratorio se puede crear una CA local y un certificado de servidor directamente en RouterOS. Esta opción no requiere generar previamente archivos `.crt` y `.key`.

1. Ir a **System > Certificates**.
2. Pulsar **+** para crear una plantilla de CA:
   - **Name:** `CA-LosRobles`
   - **Common Name:** `CA-LosRobles`
   - **Days Valid:** `365` o el periodo definido para el laboratorio.
   - **Key Usage:** seleccionar `key-cert-sign` y `crl-sign`.
   - Definir organización, país y demás campos si se requieren.
3. Pulsar **Apply** y **OK**.
4. Seleccionar `CA-LosRobles`, pulsar **Sign** y confirmar la firma como certificado raíz.
5. Crear otra plantilla con **+**:
   - **Name:** `Hotspot-LosRobles`
   - **Common Name:** el nombre DNS que se usará en el portal, por ejemplo `login.losrobles.edu`.
   - **Subject Alt. Name:** agregar `DNS:login.losrobles.edu`. Si el laboratorio accederá por IP, agregar también `IP:192.168.88.1`.
   - **Key Usage:** `digital-signature`, `key-encipherment` y `tls-server`.
   - Usar el mismo periodo de validez y un algoritmo seguro, preferiblemente SHA-256.
6. Pulsar **Apply** y **OK**.
7. Seleccionar `Hotspot-LosRobles`, pulsar **Sign**, elegir `CA-LosRobles` como **CA Certificate** y confirmar.
8. En la lista de certificados, comprobar que:
   - `CA-LosRobles` tenga la marca de autoridad.
   - `Hotspot-LosRobles` tenga clave privada y aparezca como emitido por la CA.
   - Ambos estén vigentes.
9. Seleccionar cada certificado, abrir **Set** o sus propiedades y marcar **Trusted** cuando RouterOS lo solicite para el servicio.

Los nombres de botones pueden variar ligeramente según la versión de Winbox. RouterOS elimina la plantilla después de firmarla; el certificado firmado es el que debe utilizarse.

Este flujo corresponde al administrador de certificados de RouterOS 7. Para consultar las propiedades disponibles de las plantillas, nombres alternativos, usos de clave y confianza, consulte la [documentación oficial de certificados de RouterOS](https://help.mikrotik.com/docs/spaces/ROS/pages/2555969/Certificates).

### 11.2 Asociar el certificado al Hotspot

1. Ir a **IP > Hotspot > Server Profiles**.
2. Abrir el perfil que utiliza el Hotspot.
3. En **SSL Certificate**, seleccionar `Hotspot-LosRobles`.
4. En **Login By**, activar `https` y conservar `http-chap` solo si se necesita compatibilidad durante las pruebas.
5. Pulsar **Apply** y **OK**.
6. Confirmar que el **DNS Name** del Hotspot coincide exactamente con el **Common Name** y el **Subject Alt. Name** del certificado.
7. Desde el cliente, resolver el nombre hacia `192.168.88.1` y abrir `https://login.losrobles.edu`.

Un certificado autofirmado no es confiable automáticamente para Firefox u otros clientes. Para eliminar la advertencia, seleccionar la CA `CA-LosRobles` en **System > Certificates**, utilizar **Export** y descargar únicamente el certificado público `.crt`. Instalar esa CA como autoridad de confianza en el cliente de laboratorio. Nunca exportar ni publicar la clave privada del certificado de servidor.

### 11.3 Importar un certificado externo

Si se utiliza un certificado emitido por una CA institucional o pública:

1. Subir al router, desde **Files**, el certificado y su clave privada protegida.
2. Ir a **System > Certificates > Import**.
3. Importar el certificado, la clave y la cadena intermedia si corresponde.
4. Confirmar que el certificado tenga clave privada y nombre DNS correcto.
5. Asociarlo al perfil del Hotspot como se explica arriba.

No subir certificados privados, claves, contraseñas ni archivos de importación al repositorio GitHub.

### 11.4 Ejecutar el asistente Hotspot

1. Ir a **IP > Hotspot**.
2. Pulsar **Hotspot Setup**.
3. Seleccionar `bridge-lan`.
4. Confirmar `192.168.88.1/24`.
5. Seleccionar un pool exclusivo del Hotspot, sin solaparlo con el DHCP principal.
6. Confirmar el certificado TLS importado.
7. En **DNS Name**, escribir un nombre institucional, por ejemplo `login.losrobles.edu`.
8. Crear temporalmente un usuario local de prueba.
9. Pulsar **Next** hasta terminar.

### 11.5 Activar RADIUS en Hotspot

1. En Ubuntu Desktop, configure la IP fija `192.168.88.3/24`, instale FreeRADIUS y registre `192.168.88.1` en `clients.conf` con un secreto compartido.
2. En Winbox, ir a **Radius > +**.
3. En **Address**, escribir `192.168.88.3`.
4. En **Service**, marcar `hotspot`.
5. En **Secret**, escribir exactamente el mismo secreto configurado en Ubuntu.
6. Pulsar **Apply > OK**.
7. Ir a **IP > Hotspot > Server Profiles**.
8. Abrir el perfil utilizado.
9. En la pestaña **Login**, activar **Use RADIUS**.
10. Seleccionar `https` y el método de autenticación compatible.
11. Pulsar **Apply > OK**.

En Ubuntu, valide con `sudo freeradius -XC`, `sudo systemctl status freeradius` y `radtest`. Desde el cliente, pruebe una cuenta de estudiante y otra de docente. En R-CORE compruebe **IP > Hotspot > Active** y **Log**. El firewall de Ubuntu debe permitir UDP `1812` (autenticación) y, si se usa accounting, UDP `1813`; restrinja el origen a `192.168.88.1`.

El atributo `Mikrotik-Group` puede asociar el usuario con un perfil de Hotspot si la integración y la versión de RouterOS lo soportan. Si no se devuelve ese atributo, cree usuarios de prueba en R-CORE o ajuste la respuesta RADIUS según la documentación de la versión instalada.

## 12. Perfiles de velocidad y usuarios

### 12.1 Crear perfiles Hotspot

1. Ir a **IP > Hotspot > User Profiles**.
2. Pulsar **+**.
3. Crear el perfil:
   - **Name:** `estudiante`
   - **Rate Limit:** `10M/10M`
   - **Shared Users:** `1`
4. Crear otro:
   - **Name:** `docente`
   - **Rate Limit:** `30M/30M`
   - **Shared Users:** `1`
5. Pulsar **Apply > OK** en ambos.

### 12.2 Crear usuarios locales de prueba

Para validar sin RADIUS:

1. Ir a **IP > Hotspot > Users**.
2. Pulsar **+**.
3. Crear un usuario de estudiante y asignar el perfil `estudiante`.
4. Crear un usuario docente y asignar el perfil `docente`.
5. Después de validar, deshabilitar o eliminar las cuentas temporales si la autenticación será únicamente RADIUS.

## 13. Configuración del AP MikroTik `AP-01`

### 13.1 Identidad y dirección de gestión

1. Conectarse al AP mediante Winbox.
2. Ir a **System > Identity** y establecer `AP-01`.
3. Ir a **Bridge > Bridge**, crear `bridge-ap`.
4. En **Bridge > Ports**, añadir la interfaz conectada hacia `R-CORE`.
5. Si existe interfaz inalámbrica compatible, añadirla al mismo bridge.
6. Ir a **IP > Addresses > +**:
   - **Address:** `192.168.88.2/24`
   - **Interface:** `bridge-ap`
7. Ir a **IP > Routes > +**:
   - **Dst. Address:** `0.0.0.0/0`
   - **Gateway:** `192.168.88.1`

### 13.2 Evitar doble NAT y DHCP

- No crear un DHCP Server en el AP.
- No crear una regla masquerade en el AP.
- No usar el puerto WAN como router independiente si el objetivo es bridge.
- Confirmar que los clientes reciban DHCP desde `R-CORE`.

### 13.3 Configurar la WLAN cuando exista soporte de radio

En **Wireless** o **WiFi** (según el paquete instalado):

1. Crear o editar la interfaz inalámbrica.
2. Configurar el SSID institucional.
3. Seleccionar el canal autorizado.
4. Usar ancho de canal de 20/40 MHz según el plan de espectro.
5. Crear un perfil de seguridad con **WPA3-SAE** si el hardware o la imagen lo soporta.
6. Utilizar una clave robusta y conservarla fuera de GitHub.
7. Asociar el perfil de seguridad a la interfaz inalámbrica.

Si el CHR/OpenWrt virtual no ofrece una radio inalámbrica real, el AP solo validará el bridge y la conectividad lógica. WPA3 deberá comprobarse con un AP físico compatible.

## 14. Registros y auditoría en Winbox

1. Ir a **System > Logging > Rules**.
2. Pulsar **+** y crear:
   - **Topics:** `firewall`
   - **Action:** `memory`
3. Repetir para:
   - `hotspot,account`
   - `radius`
4. Ir a **Log** para revisar eventos.

Para producción, configurar una acción de tipo **remote** hacia un servidor Syslog institucional. No registrar contraseñas ni secretos.

## 15. Validación desde Winbox y Webterm

### 15.1 Estado general

- **Interfaces:** confirmar que WAN y LAN estén `R` (running).
- **IP > DHCP Client:** comprobar estado `bound`.
- **IP > DHCP Server > Leases:** confirmar clientes activos.
- **IP > Routes:** verificar una ruta por defecto.

### 15.2 Pruebas desde Webterm

```bash
ip addr
ip route
ping -c 4 192.168.88.1
nslookup example.com 192.168.88.1
dig @8.8.8.8 example.com
iperf3 -c IP_SERVIDOR_PRUEBAS -t 30
```

Resultados esperados:

- El cliente recibe una IP del rango configurado.
- El ping al gateway es bloqueado por la política.
- El DNS se resuelve a través del router.
- El navegador es redirigido al portal HTTPS.
- El perfil estudiante se aproxima como máximo a 10 Mbps.
- El perfil docente se aproxima como máximo a 30 Mbps.

En **IP > Firewall**, revisar los contadores de reglas. En **IP > Hotspot > Active**, confirmar la sesión, usuario, dirección IP y perfil aplicado.

## 16. Lista de comprobación final

- [ ] Se cambió la contraseña administrativa.
- [ ] Se creó el respaldo inicial.
- [ ] `R-CORE` tiene WAN y LAN correctamente identificadas.
- [ ] `bridge-lan` contiene únicamente las interfaces LAN.
- [ ] El DHCP entrega direcciones sin solapamiento.
- [ ] NAT funciona únicamente hacia la WAN.
- [ ] Firewall bloquea administración desde WAN.
- [ ] ICMP al gateway queda bloqueado según el requisito.
- [ ] DNS TCP/UDP 53 es redirigido al resolutor autorizado.
- [ ] Las MAC del AP y del servidor fueron sustituidas por valores reales.
- [ ] El certificado TLS es válido y no está publicado en GitHub.
- [ ] RADIUS autentica usuarios correctamente.
- [ ] Los perfiles `estudiante` y `docente` aplican sus límites.
- [ ] El AP trabaja como bridge, sin DHCP ni doble NAT.
- [ ] Los eventos de firewall, Hotspot y RADIUS aparecen en los logs.
- [ ] Se guardó un export sanitizado sin secretos.
