# Colegio Los Robles — Red inalámbrica segura en GNS3

Propuesta técnica y guía de implementación para emular en Linux una red inalámbrica segura, segmentada y con control de calidad de servicio para el Colegio Los Robles.

## 1. Resumen ejecutivo

El colegio cuenta con más de 1.000 usuarios entre estudiantes, docentes y personal administrativo. La red actual presenta interrupciones durante las horas lectivas, saturación de los enlaces e intentos de acceso no autorizado a los servicios institucionales.

La solución propuesta utiliza **GNS3 sobre Linux** para emular un perímetro de red MikroTik RouterOS, un punto de acceso virtual y clientes de prueba. El diseño centraliza el enrutamiento, DHCP, firewall, portal cautivo, autenticación RADIUS, control de ancho de banda y auditoría.

> **Alcance de la emulación:** GNS3 valida la lógica de red y las políticas de seguridad. La radiofrecuencia, la cobertura, la interferencia y el rendimiento físico WPA3 deben validarse posteriormente con equipos inalámbricos reales.

## 2. Objetivos

### Objetivo general

Diseñar, implementar y validar una arquitectura inalámbrica segura y administrable que garantice conectividad estable, autenticación centralizada y uso equitativo del ancho de banda.

### Objetivos específicos

- Separar el acceso de estudiantes del acceso administrativo.
- Centralizar DHCP, NAT, firewall, Hotspot y políticas QoS en RouterOS.
- Proteger el portal cautivo mediante HTTPS/TLS.
- Autenticar usuarios con RADIUS/User Manager.
- Limitar el consumo a **10 Mbps** para estudiantes y **30 Mbps** para docentes.
- Registrar autenticaciones, bloqueos y eventos relevantes para auditoría.
- Dejar preparada la topología para añadir APs y roaming posteriormente.

## 3. Arquitectura propuesta

```text
                    ┌─────────────────────┐
                    │ Internet / NAT GNS3 │
                    └──────────┬──────────┘
                               │ WAN
                    ┌──────────▼──────────┐
                    │ MikroTik CHR Router │
                    │ RouterOS 7.x        │
                    │ NAT · FW · DHCP     │
                    │ Hotspot · RADIUS    │
                    └───────┬───────┬──────┘
                            │       │ LAN/Bridge
                ┌───────────▼─┐   ┌─▼────────────────┐
                │ AP virtual  │   │ Webterm / cliente │
                │ CHR/OpenWrt │   │ Firefox + CLI     │
                └───────┬─────┘   └──────────────────┘
                        │
                SSID institucional
```

### Componentes

| Componente | Tecnología | Función |
|---|---|---|
| Host | Linux | Ejecutar GNS3, Docker/VMs y herramientas de prueba |
| Orquestador | GNS3 | Crear enlaces, switches y nodos virtuales |
| Router/core | MikroTik CHR RouterOS 7.x | Gateway, NAT, firewall, DHCP, Hotspot, QoS y RADIUS |
| AP | MikroTik CHR u OpenWrt VM | Puente de acceso y emulación lógica de la WLAN |
| Cliente | Webterm Docker/VM | Navegador, terminal y pruebas de políticas |
| Identidad | User Manager/RADIUS | Autenticación y asignación de perfiles |

## 4. Plan de direccionamiento

| Uso | Red o rango | Gateway |
|---|---|---|
| LAN institucional | `192.168.88.0/24` | `192.168.88.1` |
| Estudiantes/general | `192.168.88.4–192.168.88.99` | `192.168.88.1` |
| Docentes/servidores | `192.168.88.100–192.168.88.200` | `192.168.88.1` |
| Gestión del AP (reservada) | `192.168.88.2` | `192.168.88.1` |
| Servidor interno (reservada) | `192.168.88.3` | `192.168.88.1` |

> Para una implementación física se recomienda separar estudiantes, docentes y servidores en VLAN/subredes distintas. En esta primera emulación se mantienen en una LAN común para simplificar el escenario y se diferencian mediante perfiles DHCP/RADIUS.

## 5. Requisitos previos

- Host Linux con virtualización habilitada (KVM recomendado).
- GNS3 GUI y GNS3 server instalados.
- Imagen legal de MikroTik CHR RouterOS 7.x.
- Imagen de OpenWrt o segundo CHR para el AP.
- Imagen Webterm con Firefox.
- Certificado TLS y clave privada para el Hotspot.
- Acceso administrativo inicial a RouterOS.
- Enlace WAN simulado en GNS3 para probar NAT y DNS.

## 6. Implementación paso a paso

### Paso 1 — Crear la topología en GNS3

1. Crear un proyecto llamado `colegio-los-robles`.
2. Añadir un CHR como `R-CORE`, un CHR/OpenWrt como `AP-01` y un Webterm como `CLIENTE-01`.
3. Añadir un switch Ethernet virtual para la LAN.
4. Conectar:
   - `R-CORE/WAN` al nodo NAT/Internet de GNS3.
   - `R-CORE/LAN` al switch LAN.
   - `AP-01` y `CLIENTE-01` al switch LAN.
5. Iniciar los nodos y confirmar que las interfaces aparecen en RouterOS con `/interface print`.

### Paso 2 — Configurar interfaces y direccionamiento

Identificar los nombres reales de las interfaces antes de ejecutar los comandos. En el ejemplo, `ether1` es WAN y `ether2` es LAN.

```routeros
/interface bridge
add name=bridge-lan comment="LAN Colegio Los Robles"

/interface bridge port
add bridge=bridge-lan interface=ether2

/ip address
add address=192.168.88.1/24 interface=bridge-lan comment="Gateway LAN"

/ip dhcp-client
add interface=ether1 disabled=no comment="WAN por DHCP"

/ip dns
set allow-remote-requests=yes
```

### Paso 3 — Configurar DHCP

```routeros
/ip pool
add name=pool-estudiantes ranges=192.168.88.4-192.168.88.99
add name=pool-administrativo ranges=192.168.88.100-192.168.88.200

/ip dhcp-server
add name=dhcp-lan interface=bridge-lan address-pool=pool-estudiantes lease-time=8h disabled=no

/ip dhcp-server network
add address=192.168.88.0/24 gateway=192.168.88.1 dns-server=192.168.88.1 \
    comment="Clientes LAN"
```

Las direcciones `192.168.88.2` y `192.168.88.3` quedan fuera del pool para reservarlas al AP y al servidor interno. Si se requiere entregar el pool administrativo automáticamente, debe utilizarse una segunda VLAN/subred o reservas DHCP asociadas a MAC; no se deben mezclar dos pools en la misma red sin una política clara de asignación.

### Paso 4 — Configurar NAT y firewall

Primero establecer una política mínima de protección. El orden de las reglas es importante: las conexiones establecidas deben aceptarse antes de descartar tráfico nuevo.

```routeros
/ip firewall nat
add chain=srcnat out-interface=ether1 action=masquerade \
    comment="NAT de salida a Internet"

/ip firewall filter
add chain=input action=accept connection-state=established,related \
    comment="Aceptar conexiones existentes"
add chain=input action=drop connection-state=invalid \
    comment="Descartar conexiones inválidas"
add chain=input action=accept protocol=udp dst-port=67,68 \
    in-interface=bridge-lan comment="Permitir DHCP"
add chain=input action=accept protocol=udp dst-port=53 \
    in-interface=bridge-lan comment="Permitir DNS UDP local"
add chain=input action=accept protocol=tcp dst-port=53 \
    in-interface=bridge-lan comment="Permitir DNS TCP local"
add chain=input action=drop protocol=icmp dst-address=192.168.88.1 \
    comment="Ocultar gateway frente a ping"
add chain=input action=drop in-interface=ether1 \
    comment="Bloquear acceso entrante desde WAN"
add chain=input action=drop comment="Denegar el resto del tráfico al router"

/ip firewall filter
add chain=forward action=accept connection-state=established,related \
    comment="Forward de conexiones existentes"
add chain=forward action=drop connection-state=invalid
add chain=forward action=accept in-interface=bridge-lan out-interface=ether1 \
    comment="Permitir salida LAN"
add chain=forward action=drop comment="Denegar forward no autorizado"
```

### Paso 5 — Forzar DNS seguro

El DNS tradicional utiliza los puertos TCP/UDP 53. La redirección debe aplicarse a esos puertos; un puerto aleatorio no constituye una consulta DNS estándar.

```routeros
/ip firewall nat
add chain=dstnat in-interface=bridge-lan protocol=udp dst-port=53 \
    action=redirect to-ports=53 comment="Forzar DNS UDP local"
add chain=dstnat in-interface=bridge-lan protocol=tcp dst-port=53 \
    action=redirect to-ports=53 comment="Forzar DNS TCP local"
```

Si se utiliza un resolutor externo con filtrado, se puede sustituir `redirect` por `dst-nat` hacia la IP y el puerto del servidor autorizado. Debe documentarse el proveedor, la política de privacidad y la disponibilidad del servicio.

### Paso 6 — Proteger el acceso ARP

Asignar entradas estáticas únicamente a equipos con IP fija y MAC verificadas:

```routeros
/ip arp
add address=192.168.88.2 mac-address=AA:BB:CC:DD:EE:02 \
    interface=bridge-lan comment="AP-01"
add address=192.168.88.3 mac-address=AA:BB:CC:DD:EE:03 \
    interface=bridge-lan comment="Servidor interno"
```

No se deben copiar estas MAC de ejemplo. Sustituirlas por los valores observados con `/ip arp print` y validar primero la conectividad. En una red con muchos clientes dinámicos, `arp=reply-only` debe habilitarse solo después de disponer de reservas DHCP y entradas ARP para todos los equipos autorizados.

### Paso 7 — Configurar Hotspot, TLS y RADIUS

1. Importar el certificado y la clave privada:

```routeros
/certificate import file-name=hotspot-los-robles.crt
/certificate import file-name=hotspot-los-robles.key
/certificate print
```

2. Ejecutar el asistente y seleccionar `bridge-lan`:

```routeros
/ip hotspot setup
```

Seleccionar la dirección `192.168.88.1/24`, un pool que no se solape con los pools DHCP, el certificado importado y un DNS name institucional, por ejemplo `login.losrobles.edu`.

3. Habilitar el uso de RADIUS:

```routeros
/radius
add address=127.0.0.1 service=hotspot secret="CAMBIAR_SECRET_RADIUS"

/ip hotspot profile
set [find default=yes] use-radius=yes login-by=https,http-chap
```

El secreto debe reemplazarse y mantenerse fuera del repositorio. En un entorno real, el certificado debe ser emitido por una autoridad confiable y los clientes deben resolver el nombre DNS correspondiente.

4. Crear perfiles de velocidad:

```routeros
/ip hotspot user profile
add name=estudiante rate-limit=10M/10M shared-users=1 \
    comment="Perfil estudiante"
add name=docente rate-limit=30M/30M shared-users=1 \
    comment="Perfil docente"
```

Crear los usuarios en User Manager/RADIUS y asociarlos al perfil correspondiente. Verificar la sintaxis y el método de integración según la versión exacta de RouterOS/User Manager instalada.

### Paso 8 — Configurar el AP virtual

- Asignar a `AP-01` la IP de gestión `192.168.88.2/24` y gateway `192.168.88.1`.
- Crear un bridge entre la interfaz LAN y la interfaz inalámbrica virtual.
- Desactivar DHCP/NAT en el AP para evitar doble NAT.
- Configurar el SSID institucional, canal y ancho de canal de **20/40 MHz**.
- Seleccionar WPA3-SAE si la imagen virtual soporta el modo inalámbrico. Si la plataforma solo ofrece bridge Ethernet, documentar WPA3 como requisito de la futura capa física, no como una capacidad realmente emulada.
- Utilizar una clave robusta y no almacenarla en este repositorio.

### Paso 9 — Habilitar auditoría

```routeros
/system logging
add topics=firewall action=memory
add topics=hotspot,account action=memory
add topics=radius action=memory

/log print
```

Para producción, enviar los registros a un syslog remoto con retención, control de acceso y sincronización NTP. No guardar contraseñas ni secretos en los logs.

## 7. Protocolo de pruebas

Registrar fecha, usuario, IP, resultado esperado, resultado observado y evidencia (captura o salida de consola).

### 7.1 Firewall del gateway

Desde Webterm:

```bash
ping -c 4 192.168.88.1
```

**Resultado esperado:** timeout o paquetes filtrados. La administración debe probarse desde una interfaz y una cuenta de gestión autorizadas, no mediante ICMP.

### 7.2 DHCP y conectividad

```bash
ip addr
ip route
nslookup example.com 192.168.88.1
```

**Resultado esperado:** dirección dentro del pool correcto, gateway `192.168.88.1` y resolución DNS funcional.

### 7.3 Redirección DNS

```bash
dig @8.8.8.8 example.com
```

En RouterOS revisar contadores:

```routeros
/ip firewall nat print stats
```

**Resultado esperado:** la consulta TCP/UDP 53 es interceptada y procesada por el resolutor autorizado. No debe aceptarse como prueba DNS un puerto arbitrario que no sea 53, salvo que se documente explícitamente un protocolo alternativo.

### 7.4 Portal cautivo y autenticación

1. Abrir Firefox en Webterm.
2. Navegar a un sitio HTTP de prueba.
3. Confirmar la redirección al portal HTTPS.
4. Iniciar sesión con un usuario de prueba de estudiante.
5. Repetir con un usuario docente.
6. Verificar en `/log print` los eventos de autenticación.

**Resultado esperado:** usuarios válidos acceden con su perfil; credenciales inválidas son rechazadas y registradas.

### 7.5 QoS

Ejecutar un test de velocidad o `iperf3` contra un servidor de prueba controlado:

```bash
iperf3 -c IP_SERVIDOR_PRUEBAS -t 30
```

**Resultado esperado:** el perfil estudiante no supera aproximadamente 10 Mbps y el perfil docente no supera aproximadamente 30 Mbps. Repetir varias veces y considerar la sobrecarga del laboratorio; los límites son máximos configurados, no una garantía de velocidad mínima.

### 7.6 Seguridad negativa

- Intentar acceder a la administración desde WAN.
- Usar credenciales inválidas.
- Intentar resolver DNS directamente sin pasar por el gateway.
- Revisar que el tráfico no autorizado aparezca en los logs.

## 8. Criterios de aceptación

- [ ] Todos los nodos arrancan y la topología está documentada en GNS3.
- [ ] El cliente recibe una dirección del rango esperado.
- [ ] El NAT permite salida a Internet sin exponer la administración.
- [ ] El ping al gateway es bloqueado según la política.
- [ ] Las consultas DNS son forzadas al resolutor autorizado.
- [ ] El portal cautivo utiliza HTTPS/TLS válido.
- [ ] RADIUS autentica y asigna los perfiles correctos.
- [ ] El perfil estudiante queda limitado a 10/10 Mbps.
- [ ] El perfil docente queda limitado a 30/30 Mbps.
- [ ] Los eventos relevantes aparecen en los registros.
- [ ] No hay contraseñas, claves privadas ni secretos en el repositorio.

## 9. Operación, ampliación y mantenimiento

### Roaming y crecimiento

Para añadir APs, reutilizar el mismo servicio RADIUS y definir una estrategia de canales no solapados. En una implementación real se deben separar las VLAN de estudiantes, docentes, servidores y gestión, además de utilizar un controlador o una solución de roaming compatible.

### Copias de seguridad y cambios

Antes de cada cambio:

```routeros
/export file=backup-antes-del-cambio
/system backup save name=backup-binario-antes-del-cambio
```

Guardar los respaldos fuera del repositorio y protegerlos con control de acceso. Probar la restauración en un laboratorio antes de aplicarla en producción.

### Riesgos y mitigaciones

| Riesgo | Mitigación |
|---|---|
| Recursos insuficientes del host | Asignar RAM/CPU según la carga y apagar nodos no usados |
| Certificado TLS inválido | Usar nombre DNS estable y certificado confiable |
| Doble NAT en el AP | Operar el AP en modo bridge |
| Solapamiento de pools | Reservar IPs de infraestructura y documentar rangos |
| Falsa sensación de WPA3 | Validar WPA3 con hardware o una VM que soporte radio virtual |
| Pérdida de trazabilidad | Centralizar logs y sincronizar hora con NTP |

## 10. Entregables finales

1. Proyecto GNS3 exportado (`.gns3project`).
2. Este documento `README.md`.
3. Diagrama de topología y tabla de direccionamiento.
4. Export sanitizado de la configuración RouterOS.
5. Matriz de pruebas con evidencias.
6. Informe de resultados, limitaciones y recomendaciones para producción.

## 11. Conclusión

La arquitectura propuesta aborda la inestabilidad, la saturación y los riesgos de acceso no autorizado mediante una administración centralizada, autenticación RADIUS, portal cautivo TLS, firewall, DNS controlado, límites de velocidad y auditoría. La emulación permite validar la lógica antes de invertir en infraestructura física; la validación de cobertura, capacidad radioeléctrica y WPA3 debe completarse en una segunda fase con equipos reales.
