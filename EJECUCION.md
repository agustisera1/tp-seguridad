# Bitácora de ejecución — TP1 Seguridad

Registro paso a paso (GUI + CLI) de lo que se va configurando en `RESOLUCION.pkt`, fase por fase.
Es el "cómo, en orden" — complementa a `PLAN_FASES.md` (que es el "qué, y en qué fase").

## Formato de cada tarea

Cada tarea sigue esta estructura:

- **Objetivo:** qué se logra con esta tarea.
- **Para qué:** por qué hace falta (conecta con el concepto de red involucrado).
- **Pasos:** instrucciones concretas en la GUI de Packet Tracer y/o comandos CLI listos para pegar.
- **Verificación:** cómo confirmar que quedó bien.

Marcá `[x]` cuando lo hayas ejecutado y confirmado en Packet Tracer.

## Cómo identificar un puerto antes de pegar un comando

Los nombres de puerto (`FastEthernet0/1`, `GigabitEthernet0/1`, `Ethernet0/0`...) dependen de cómo
cableaste vos en tu `.pkt`. Antes de correr un comando que menciona un puerto, confirmá el nombre real:

1. En la vista **Logical**, pasá el mouse sobre el cable que te interesa: Packet Tracer muestra
   "Dispositivo:Puerto" en ambos extremos.
2. O entrá al dispositivo → **CLI** → `show ip interface brief` (routers/ASA) o
   `show interfaces status` (switches) para ver qué puertos están "up" y a qué corresponden.

Esta técnica se repite en todas las fases: siempre verificá el puerto real antes de pegar.

---

## Fase 1 — Topología física + direccionamiento base

**Objetivo de la fase** (`PLAN_FASES.md`): topología cableada, IP/máscara/gateway en todos los
hosts, VLANs 10/20/30 + trunk en SW-LAN, puerto de acceso en SW-DMZ.
**Estado al cierre:** enlaces up; ping dentro de cada segmento donde ya existe un gateway configurado.

**Nota de alcance (decisión tomada en esta sesión):** las interfaces internas de la ASA que van a
ser gateway de VLAN 10/20/30 dependen de si el trunk ASA↔SW-LAN funciona en este modelo — ese es
el *checkpoint de contingencia* que el plan reserva para la Fase 2. Por eso acá solo se configuran
`outside` y `dmz` en la ASA (no dependen de trunk). El lado ASA de las VLANs internas queda para
Fase 2. Consecuencia: los pings de las PCs a su gateway (.10.1/.20.1/.30.1) **no van a andar
todavía** — se validan al cerrar Fase 2.

- [x] Tarea 1 — WEB-SERVER: IP fija en la DMZ
- [x] Tarea 2 — SW-DMZ: puertos de acceso (WEB-SERVER y enlace a la ASA)
- [x] Tarea 3 — Router ISP: IP en la interfaz hacia la ASA
- [x] Tarea 4 — ASA 5505: interfaces `outside` y `dmz`
- [x] Tarea 5 — SW-LAN: VLANs 10/20/30 + puertos de acceso + trunk hacia la ASA
- [x] Verificación de cierre de Fase 1

### Tarea 1 — WEB-SERVER: IP fija en la DMZ

**Objetivo:** asignar `192.168.40.10 /24`, gateway `192.168.40.1` al WEB-SERVER.
**Para qué:** es el host final de la DMZ; sin IP no hay nada que publicar ni que probar en las
fases siguientes.

**Pasos (GUI, no tiene CLI de Cisco — es un Server-PT):**
1. Doble clic en **WEB-SERVER**.
2. Pestaña **Desktop** → **IP Configuration**.
3. Elegí **Static**.
4. IP Address: `192.168.40.10` — Subnet Mask: `255.255.255.0` — Default Gateway: `192.168.40.1`.
5. Cerrá la ventana. El puerto del server debería verse verde en el lienzo (si está bien cableado).

**Verificación:** Desktop → **Command Prompt** → `ipconfig` te tiene que devolver esa IP.

### Tarea 2 — SW-DMZ: puertos de acceso

**Objetivo:** dejar explícitos (aunque ya vengan así por defecto) los puertos de SW-DMZ: uno hacia
el WEB-SERVER y otro hacia la ASA, ambos en modo acceso, VLAN 1 (la DMZ es una sola red, no
necesita VLANs adicionales).
**Para qué:** práctica prolija — documentar el puerto con `description` ayuda a vos mismo (y a mí)
a revisar más adelante.

**Pasos (CLI, doble clic en SW-DMZ → CLI):**
```
enable
configure terminal
hostname SW-DMZ

! Puerto hacia el WEB-SERVER (reemplazá FastEthernet0/1 por el puerto real)
interface FastEthernet0/1
 switchport mode access
 switchport access vlan 1
 no shutdown
 description Acceso a WEB-SERVER
exit

! Puerto hacia la ASA dmz (reemplazá FastEthernet0/2 por el puerto real)
interface FastEthernet0/2
 switchport mode access
 switchport access vlan 1
 no shutdown
 description Enlace a ASA (dmz)
exit

end
write memory
```
- `switchport mode access` / `switchport access vlan 1`: fija el puerto a una sola VLAN (la 1, la
  que viene por defecto) — no hace falta trunk acá porque del lado DMZ no hay múltiples VLANs.
- `no shutdown`: los puertos de switch a veces quedan administrativamente apagados; esto los prende.
- `write memory`: guarda la configuración corriendo como configuración de arranque (si no, se
  pierde si se reinicia el dispositivo simulado).

**Verificación:** `show vlan brief` (los dos puertos deben figurar en VLAN 1) y
`show interfaces status` (deben decir "connected").

### Tarea 3 — Router ISP: IP en la interfaz hacia la ASA

**Objetivo:** asignar `200.10.10.1 /30` a la interfaz del Router ISP conectada a la ASA (`outside`).
**Para qué:** es el otro extremo del enlace WAN; sin esto la ASA no tiene con quién hablar hacia
"afuera". (La salida real a Internet/nube se resuelve en Fase 3 — hoy solo este tramo.)

**Pasos (CLI, doble clic en Router ISP → CLI):**
```
enable
configure terminal
hostname ISP

! Reemplazá GigabitEthernet0/0 por el puerto real conectado a la ASA
interface GigabitEthernet0/0
 ip address 200.10.10.1 255.255.255.252
 no shutdown
 description Enlace a ASA outside
exit

end
write memory
```
- `/30` (máscara `255.255.255.252`) da solo 2 IPs utilizables — perfecto para un enlace
  punto a punto de solo dos equipos (Router ISP y ASA).
- `no shutdown`: en los routers Cisco, las interfaces vienen apagadas por defecto; hay que
  levantarlas explícitamente.

**Verificación:** `show ip interface brief` → la interfaz debe figurar `up / up` con esa IP.

### Tarea 4 — ASA 5505: interfaces `outside` y `dmz`

**Objetivo:** configurar las dos interfaces de la ASA que no dependen del trunk: `outside` (hacia
Router ISP) y `dmz` (hacia SW-DMZ).
**Para qué:** son el perímetro de la red — todo lo que entra o sale por Internet, y todo lo que
llega al WEB-SERVER, pasa por acá. La ASA 5505 asigna `security-level` a cada interfaz (0–100,
más alto = más confiable) y por defecto solo deja pasar tráfico de mayor a menor nivel.

**Cómo son las interfaces en una ASA 5505 (a diferencia de un router):** las bocas físicas
(`Ethernet0/0`, `Ethernet0/1`...) son puertos de switch, no interfaces ruteadas directamente. La
IP, el `nameif` y el `security-level` se configuran sobre una interfaz lógica `Vlan<N>` interna de
la ASA; después la boca física se asigna a esa VLAN con `switchport access vlan <N>`. Es un
concepto nuevo pero se usa igual en toda la ASA 5505.

**Pasos (CLI, doble clic en ASA → CLI):**
```
enable
configure terminal
hostname ASA

! ---- outside: hacia el Router ISP / Internet ----
interface Vlan2
 nameif outside
 security-level 0
 ip address 200.10.10.2 255.255.255.252
 no shutdown
exit

! Reemplazá Ethernet0/0 por el puerto real conectado al Router ISP
interface Ethernet0/0
 switchport access vlan 2
 no shutdown
exit

! ---- dmz: hacia el SW-DMZ / WEB-SERVER ----
interface Vlan3
 nameif dmz
 security-level 50
 ip address 192.168.40.1 255.255.255.0
 no shutdown
exit

! Reemplazá Ethernet0/1 por el puerto real conectado a SW-DMZ
interface Ethernet0/1
 switchport access vlan 3
 no shutdown
exit

end
write memory
```
- `security-level 0` en `outside`: la interfaz menos confiable (así es Internet).
- `security-level 50` en `dmz`: confianza intermedia — más que Internet, menos que la LAN interna
  (que va a quedar en 100 cuando configuremos VLAN 10/20/30 en Fase 2).
- Las VLANs internas 10/20/30 (interfaz `inside`) **no se tocan hoy** — quedan para la Fase 2, ver
  la nota de alcance al principio de esta sección.

**Verificación:**
- `show interface ip brief` (o `show nameif`) en la ASA: `outside` y `dmz` deben figurar `up` con
  sus IPs.
- Desde Router ISP: `ping 200.10.10.2` → debe responder.
- Desde la ASA: `ping outside 200.10.10.1` y `ping dmz 192.168.40.10` → deben responder (el nombre
  de interfaz después de `ping` le indica a la ASA por dónde salir, útil cuando hay varias).

**Troubleshooting real (apareció en ejecución):** al configurar `dmz` puede saltar
`ERROR: This license does not allow configuring more than 2 interfaces with nameif...`. Causa: la
licencia **Base** de la ASA 5505 solo permite 2 interfaces `nameif` completas (+ 1 restringida con
`no forward`), y la ASA trae de fábrica `Vlan1` con `nameif inside` ya cargado — sumado a `outside`
ya configurada, `dmz` sería la 3ra y choca con el límite. Solución aplicada: liberar el `inside` de
fábrica (no lo usamos, las VLANs internas van en Fase 2) antes de terminar `dmz`:
```
interface Vlan1
 no nameif
 no ip address
exit
```
Después sí, retomar `interface Vlan3` con `nameif dmz` normalmente.

**Aviso a futuro:** esto es el mismo checkpoint de contingencia de `PLAN_FASES.md` (Fase 2) pero
manifestado antes: con `outside` + `dmz` + 3 VLANs internas nombradas serían 5 interfaces
`nameif`, y la licencia Base tiene tope ~3 (2 libres + 1 restringida). En Fase 2 vamos a tener que
decidir el pivot (switch L3 / router interno para inter-VLAN) que el plan ya preveía.

**Ojo con un atajo que no funciona:** poner `security-level` e `ip address` en Vlan3 sin `nameif`
"parece" andar (no tira error, y `show interface ip brief` muestra la IP con estado `up/up`), pero
la interfaz **no reenvía tráfico** sin nombre — el ping daba 0% de éxito. `nameif` es obligatorio,
no se puede saltear. Confirmado y resuelto: se liberó `Vlan1` (`no nameif` + `no ip address`) y
recién ahí `interface Vlan3 → nameif dmz` quedó realmente operativa (ping a WEB-SERVER OK).

### Tarea 5 — SW-LAN: VLANs 10/20/30 + puertos de acceso + trunk hacia la ASA

**Objetivo:** crear las 3 VLANs internas, asignar el puerto de cada PC a su VLAN, y dejar el
puerto que sube a la ASA como trunk 802.1Q permitiendo esas 3 VLANs.
**Para qué:** cada VLAN es la "red lógica" de un segmento (Administración/Sistemas/Usuarios); el
trunk es el único cable que va a llevar el tráfico de las tres VLANs a la vez hacia la ASA
(etiquetado por VLAN) en vez de necesitar un cable por VLAN.

**Pasos (CLI, doble clic en SW-LAN → CLI):**
```
enable
configure terminal
hostname SW-LAN

vlan 10
 name ADMINISTRACION
exit
vlan 20
 name SISTEMAS
exit
vlan 30
 name USUARIOS
exit

! Reemplazá los puertos por los reales de cada PC
interface FastEthernet0/1
 switchport mode access
 switchport access vlan 10
 no shutdown
 description PC-ADM1
exit

interface FastEthernet0/2
 switchport mode access
 switchport access vlan 20
 no shutdown
 description PC-SIS1
exit

interface FastEthernet0/3
 switchport mode access
 switchport access vlan 30
 no shutdown
 description PC-USR1
exit

! Reemplazá GigabitEthernet0/1 por el puerto real hacia la ASA
interface GigabitEthernet0/1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30
 no shutdown
 description Trunk hacia ASA (inside)
exit

end
write memory
```
- `vlan 10` / `name ...`: crea la VLAN y le pone un nombre descriptivo (no cambia el funcionamiento,
  ayuda a leer `show vlan brief`).
- `switchport mode access` + `switchport access vlan X`: el puerto de cada PC queda "encerrado" en
  una sola VLAN.
- `switchport trunk encapsulation dot1q`: en switches 2960 que soportan más de un tipo de
  encapsulado hay que elegirlo explícito. **Si Packet Tracer te tira error en esta línea**, es
  porque tu modelo de 2960 solo soporta dot1q y el comando no existe — seguí directo con
  `switchport mode trunk`, no es un problema.
- `switchport trunk allowed vlan 10,20,30`: sin esto el trunk deja pasar *todas* las VLANs por
  defecto; lo explicitamos a las tres que nos importan.

**Importante — qué esperar hoy:** el otro extremo de este cable (la ASA, puerto `Ethernet0/x`) va
a seguir configurado como puerto de **acceso** hasta la Fase 2 (ver Tarea 4). Eso significa que el
enlace físico va a quedar arriba (cumple el "enlaces up" de esta fase), pero el tráfico *etiquetado*
de las VLANs 10/20/30 todavía no va a cruzar hacia la ASA — recién se valida en Fase 2, que es
justo donde está el checkpoint de "¿esta ASA soporta trunk?".

**Verificación:**
- `show vlan brief` → los 3 puertos de PC deben figurar cada uno en su VLAN.
- `show interfaces trunk` → el puerto hacia la ASA debe figurar como trunk, VLANs permitidas 10,20,30.

### Verificación de cierre de Fase 1

Con las 5 tareas hechas, este es el chequeo completo:

| Prueba | Desde | Comando | Resultado esperado |
|---|---|---|---|
| Enlace WAN | Router ISP | `ping 200.10.10.2` | Responde (ASA outside) |
| Enlace DMZ | ASA | `ping dmz 192.168.40.10` | Responde (WEB-SERVER) |
| Gateway DMZ | WEB-SERVER | `ping 192.168.40.1` (Command Prompt) | Responde (ASA dmz) |
| VLANs en SW-LAN | SW-LAN | `show vlan brief` | Cada PC en su VLAN (10/20/30) |
| Trunk en SW-LAN | SW-LAN | `show interfaces trunk` | Puerto hacia ASA en trunk |
| Interfaces ASA | ASA | `show interface ip brief` | `outside` y `dmz` up con sus IPs |

**Lo que todavía NO va a funcionar (a propósito, es de Fase 2):** ping de PC-ADM1/PC-SIS1/PC-USR1
a su gateway (`192.168.10.1` / `.20.1` / `.30.1`), porque el lado ASA de esas VLANs no está
configurado aún.

Cuando tengas esto corrido, pasame capturas o el `show running-config` / resultados de los pings
que puedas y lo valido contra este checklist antes de pasar a Fase 2.

**Cierre confirmado:** `ping 200.10.10.2` desde Router ISP → 100% (5/5). `ping 192.168.40.1` desde
WEB-SERVER → 100% (4/4, 0% loss). VLANs, trunk e interfaces de ASA ya verificados en las tareas
anteriores. **Fase 1 completa.**

**Errata post-cierre (2026-09-14):** la Tarea 4 (ASA `outside`/`dmz`) quedó **superada**. Al migrar
a Router 4331 en Fase 2 (por el límite de licencia de la ASA, ver sección "Migración" abajo), esas
dos interfaces se reconfiguran en el nuevo dispositivo. El resto de Fase 1 (Tareas 1, 2, 3, 5)
sigue vigente sin cambios — no hace falta tocarlas.

---

## Fase 2 — Migración a Router 4331 + Inter-VLAN

**Objetivo de la fase** (`PLAN_FASES.md`): reemplazar la ASA 5505 por el Router Cisco 4331 en el
rol de perímetro, y dejar las 3 VLANs internas con su propio gateway (inter-VLAN real).
**Estado al cierre:** las VLANs se pingean entre sí y alcanzan la DMZ; el enlace WAN sigue up.

**Motivo de la migración (por qué ya no es una ASA):** ver `FIREWALL_LICENSE_ISSUE.md` — límite de
licencia de la ASA 5505 y el tradeoff router-vs-firewall dedicado.

- [x] Tarea 2 — Reemplazar la ASA 5505 por el Router 4331 en el lienzo (GUI)
- [x] Tarea 3 — Router 4331: interfaces `outside`, `dmz` y subinterfaces VLAN10/20/30 (CLI)
- [x] Tarea 4 — Confirmar el trunk de SW-LAN apunta al puerto correcto del 4331
- [x] Verificación de cierre de Fase 2

### Tarea 2 — Reemplazar la ASA 5505 por el Router 4331 (GUI)

**Pasos:**
1. En el lienzo, hacé click en cada uno de los 3 cables conectados a la ASA 5505 (hacia Router
   ISP, hacia SW-DMZ, hacia SW-LAN) y eliminalos (seleccionar el cable → tecla `Delete`).
2. Seleccioná la ASA 5505 y eliminala (`Delete`). Si querés conservar referencia de su config
   vieja, ya está documentada en la Tarea 4 de Fase 1 — no hace falta guardar nada más.
3. Panel de dispositivos → categoría **Network Devices → Router** → buscá el modelo **4331** y
   arrastralo al lienzo, en el lugar donde estaba la ASA.
4. **Antes de cablear, revisá los puertos disponibles:** doble clic en el 4331 → pestaña
   **Physical**. Los 3 puertos Gigabit onboard (`GigabitEthernet0/0/0`, `0/0/1`, `0/0/2`) son
   slots **SFP** ubicados juntos en la **sección amarilla, arriba a la izquierda del chasis** — no
   sirven hasta que les pongas un transceiver. Los que veas vacíos: apagá el equipo (interruptor de
   power a la derecha del chasis), arrastrá el módulo **GLC-T** (SFP de cobre, lista de módulos a
   la izquierda) a cada slot vacío de esa sección, y volvé a prenderlo. Con eso tenés los
   **3 puertos** que necesitás: uno a Router ISP, uno a SW-DMZ, uno a SW-LAN. (Si tu versión de PT
   ya trae alguno activo por defecto, no hace falta tocar ese — solo completá los que falten.)
5. Cableá con **Copper Straight-Through** (o el que corresponda) los 3 enlaces: 4331↔Router ISP,
   4331↔SW-DMZ, 4331↔SW-LAN. Fijate el nombre real de cada puerto pasando el mouse sobre el cable.

### Tarea 3 — Router 4331: `outside`, `dmz` y subinterfaces VLAN10/20/30 (CLI)

**Objetivo:** dejar el 4331 con las mismas IPs que tenía la ASA en `outside`/`dmz`, más un gateway
propio por VLAN.
**Para qué:** en un router no existe el concepto de interfaz `Vlan<N>` con `nameif` de la ASA —
cada interfaz física o subinterfaz simplemente tiene una IP y ya reenvía tráfico; no hace falta
nombrarla ni asignarle un "security-level" para que funcione.

**Pasos (CLI, doble clic en el Router 4331 → CLI). Estos nombres de puerto ya son los reales según
cableaste vos: `0/0/0`→SW-LAN, `0/0/1`→SW-DMZ, `0/0/2`→Router ISP. Si en tu `.pkt` quedó distinto,
ajustá los nombres antes de pegar:**
```
enable
configure terminal
hostname R-PERIMETRO

! ---- outside: hacia Router ISP (antes en la ASA) ----
interface GigabitEthernet0/0/2
 ip address 200.10.10.2 255.255.255.252
 no shutdown
 description Enlace a Router ISP (outside)
exit

! ---- dmz: hacia SW-DMZ / WEB-SERVER (antes en la ASA) ----
interface GigabitEthernet0/0/1
 ip address 192.168.40.1 255.255.255.0
 no shutdown
 description Enlace a SW-DMZ (dmz)
exit

! ---- trunk hacia SW-LAN: una subinterfaz por VLAN ----
interface GigabitEthernet0/0/0
 no shutdown
 description Trunk hacia SW-LAN (inter-VLAN)
exit

interface GigabitEthernet0/0/0.10
 encapsulation dot1Q 10
 ip address 192.168.10.1 255.255.255.0
 description Gateway VLAN10 ADMINISTRACION
exit

interface GigabitEthernet0/0/0.20
 encapsulation dot1Q 20
 ip address 192.168.20.1 255.255.255.0
 description Gateway VLAN20 SISTEMAS
exit

interface GigabitEthernet0/0/0.30
 encapsulation dot1Q 30
 ip address 192.168.30.1 255.255.255.0
 description Gateway VLAN30 USUARIOS
exit

end
write memory
```
- `encapsulation dot1Q <N>`: le dice a la subinterfaz qué etiqueta de VLAN escuchar/poner en los
  paquetes — es el equivalente en IOS al `switchport access vlan N` que usaba la ASA, pero acá las
  3 conviven en el mismo puerto físico porque es un trunk real.
- No hace falta `nameif` ni `security-level`: en IOS un router reenvía entre todas sus interfaces
  por defecto (a diferencia de la ASA, que bloquea por default salvo que el nivel lo permita). Esto
  es justo la diferencia que se retoma en Fase 4 con las ACLs.

### Tarea 4 — Confirmar el trunk de SW-LAN

**Objetivo:** verificar que el puerto trunk de SW-LAN (configurado en Fase 1, Tarea 5) sigue
apuntando al puerto correcto ahora que el otro extremo es el 4331 y no la ASA.
**Pasos:** en SW-LAN, `show interfaces trunk` — confirmá que el puerto conectado al 4331 sigue
en modo trunk con VLANs 10,20,30 permitidas. Si por el recableo terminó en un puerto físico
distinto, repetí ahí la config de trunk de la Tarea 5 de Fase 1 (`switchport trunk encapsulation
dot1q` / `switchport mode trunk` / `switchport trunk allowed vlan 10,20,30`).

### Verificación de cierre de Fase 2

| Prueba | Desde | Comando | Resultado esperado |
|---|---|---|---|
| Enlace WAN | Router ISP | `ping 200.10.10.2` | Responde (Router 4331, outside) |
| Enlace DMZ | Router 4331 | `ping 192.168.40.10` | Responde (WEB-SERVER) |
| Gateway VLAN10 | PC-ADM1 | `ping 192.168.10.1` | Responde |
| Gateway VLAN20 | PC-SIS1 | `ping 192.168.20.1` | Responde |
| Gateway VLAN30 | PC-USR1 | `ping 192.168.30.1` | Responde |
| Inter-VLAN | PC-ADM1 | `ping 192.168.20.10` (PC-SIS1) | Responde — confirma comunicación entre VLAN (consigna 3) |
| Inter-VLAN | PC-ADM1 | `ping 192.168.30.10` (PC-USR1) | Responde |
| Interfaces 4331 | Router 4331 | `show ip interface brief` | `outside`, `dmz` y las 3 subinterfaces up/up |

Cuando corras esto, pasame los resultados (capturas o texto) y lo valido contra este checklist
antes de pasar a Fase 3.

**Cierre confirmado:** las 3 PCs pingean su gateway de VLAN y se pingean entre sí (inter-VLAN OK),
el enlace WAN sigue up y el 4331 alcanza el WEB-SERVER por `dmz`. **Fase 2 completa.**

---

## Fase 3 — Salida a Internet y publicación de DMZ

**Objetivo de la fase** (`PLAN_FASES.md`): que un cliente "Externo" (fuera de la organización)
pueda llegar al WEB-SERVER publicado, solo por HTTPS y FTP (consigna punto 4, fila "Externos").
**Estado al cierre:** PC-EXT (más allá del Router ISP) navega HTTPS y sube un archivo por FTP al
WEB-SERVER publicado; cualquier otro puerto/servicio queda bloqueado.

**Nota de alcance (leer antes de arrancar):** la imagen de la consigna no dibuja ningún PC más allá
del Router ISP — solo la nube "Internet". Pero sin un cliente real ahí, no hay forma de generar
tráfico HTTPS/FTP de verdad para las capturas que pide la entrega (un ping no alcanza). Se agrega
**PC-EXT**, un PC genérico (misma categoría "PC" que ya contempla la lista de dispositivos de la
consigna, no un equipo de red nuevo) conectado a la **Nube Internet que ya está en la topología**.
El tramo Router ISP↔Nube↔PC-EXT necesita una subred que la consigna no da (no hay nada especificado
más allá del Router ISP): se usa `100.100.100.0/30`, elegida arbitrariamente solo para poder probar
— no es parte del direccionamiento oficial de `PLAN_FASES.md`.

Todo lo demás de esta fase (NAT, ruta por defecto, ACL de entrada) usa **solo** los equipos ya
presentes (Router ISP y Router 4331) — no se agrega hardware de red nuevo.

**Sobre las ACL de esta fase:** acá solo se restringe el tráfico que **entra desde afuera**
(Externos → DMZ), que es lo mínimo para no dejar el NAT abierto a cualquier puerto. La matriz
completa de ACL para Administración/Sistemas/Usuarios es Fase 4.

- [x] Tarea 1 — PC-EXT: cablear hasta la Nube Internet vía un modem DSL (GUI)
- [x] Tarea 2 — Router ISP: IP en el puerto hacia la Nube (CLI)
- [x] Tarea 3 — PC-EXT: IP fija (GUI)
- [x] Tarea 4 — Router 4331: ruta por defecto hacia Router ISP (CLI)
- [x] Tarea 5 — Router 4331: NAT estático del WEB-SERVER (CLI)
- [x] Tarea 6 — Router 4331: ACL de entrada en `outside` (CLI)
- [x] Tarea 7 — WEB-SERVER: activar HTTP/HTTPS/FTP (GUI)
- [x] Verificación de cierre de Fase 3

### Tarea 1 — PC-EXT: cablear hasta la Nube Internet vía un modem DSL (GUI)

**Objetivo:** colgar PC-EXT "más allá" de la Nube "INTERNET" que **ya está** en la topología (viene
de Fase 1, cableada a `Router ISP:GigabitEthernet0/0/0` por su puerto `Ethernet6`).
**Para qué:** simular un cliente fuera de la organización que solo puede llegar al WEB-SERVER
publicado por HTTPS/FTP — hace falta tráfico real (no alcanza un ping) para las capturas de
"accesos permitidos y bloqueados" que pide la entrega.

**Intento fallido (queda documentado para no repetirlo):** agregarle a la Nube un segundo módulo
Ethernet (`PT-CLOUD-NM-1CFE` → puerto `FastEthernet8`) y buscar un "Port Mapping" genérico
Ethernet↔Ethernet en `Config`. **Esa función no existe en Packet Tracer.** Se probó también con una
Nube `PT-Empty` armada desde cero y pasa lo mismo. El `Cloud-PT` solo sabe *traducir* un puerto de
tecnología WAN (Modem → DSL, Coaxial → Cable, Serial → Frame Relay) hacia un puerto Ethernet — nunca
puentea dos puertos Ethernet entre sí directamente. Cada puerto Ethernet de la nube es el "lado
cliente" de una de esas tecnologías, no un puerto de switch genérico. Confirmado contra la
documentación oficial de Packet Tracer Tutorials y foros de Cisco Community (referencias al pie).

**Solución que funciona:** agregar un dispositivo **DSL-Modem-PT** real entre PC-EXT y la Nube, y
activar el mapeo DSL↔Ethernet que la propia Nube ya sugiere por defecto.

**Pasos:**
1. (Si habías hecho el intento fallido) Borrá el cable PC-EXT↔`FastEthernet8` y, si querés
   prolijidad, apagá la Nube y sacale ese módulo — ya no se usa.
2. Arrastrá un **PC** nuevo al lienzo, llamalo **PC-EXT** (si todavía no lo habías creado).
3. Panel de dispositivos → **Network Devices → WAN Emulation** → arrastrá un **DSL-Modem-PT** al
   lienzo, cerca de PC-EXT.
4. Cableá **PC-EXT ↔ DSL-Modem-PT** con **Copper Straight-Through** (puerto Ethernet del modem,
   confirmá el nombre real pasando el mouse sobre el cable).
5. Cableá **DSL-Modem-PT ↔ Nube "INTERNET"**, puerto `Modem4` de la Nube. Si **Copper** no engancha
   (es un puerto de línea telefónica, no RJ45), usá el cable tipo **Phone** de la paleta de
   Connections.
6. Doble clic en la Nube → **Config → CONNECTIONS → DSL**. Los desplegables ya muestran por
   defecto `Modem4 <-> Ethernet6` (el lado que va a PC-EXT vía el modem, y el que ya va a Router
   ISP) — con esos dos valores seleccionados, tocá **Add**. Debe aparecer una fila nueva en la
   tabla `From Port / To Port`: esa fila es la que activa el puente DSL↔Ethernet dentro de la Nube.

**Verificación:** con Router ISP y PC-EXT ya direccionados (Tareas 2 y 3), `ping 100.100.100.1`
desde PC-EXT responde. **Confirmado, Tarea 1 cerrada.**

**Referencias:** [Packet Tracer Tutorials — Devices and Modules](https://tutorials.ptnetacad.net/help/default/devicesAndModules_others.htm),
[Cisco Community — Emulate Internet with PT-Cloud](https://community.cisco.com/t5/vpn/emulate-internet-with-pt-cloud-in-packet-tracer/td-p/1563991),
[Cisco Community — Cable port mapping](https://community.cisco.com/t5/online-tools-and-resources/create-a-simple-network-using-packet-tracer-cannot-add-second/m-p/4522965/highlight/true).

### Tarea 2 — Router ISP: activar y direccionar el puerto hacia la nube (CLI)

**Pasos (CLI, doble clic en Router ISP → CLI):**
```
enable
configure terminal

interface GigabitEthernet0/0/0
 ip address 100.100.100.1 255.255.255.252
 no shutdown
 description Enlace hacia Internet (PC-EXT, vía Nube)
exit

end
write memory
```
- Esta interfaz **ya existe y ya está cableada** a la nube desde Fase 1 — solo estaba apagada
  (`shutdown`) y sin IP. No es necesario tocar `GigabitEthernet0/0/1` (esa es `200.10.10.1/30`,
  el enlace hacia el 4331, ya configurado y andando).
- No hace falta ninguna ruta extra en Router ISP: como las dos redes (`200.10.10.0/30` hacia el
  4331 y `100.100.100.0/30` hacia PC-EXT) están **directamente conectadas** a sus interfaces, el
  router ya sabe llegar a ambas solo. Esto es a propósito: así Router ISP nunca necesita conocer
  las redes privadas internas (`192.168.x.x`) — tal cual pasaría con un ISP real, que jamás rutea
  direcciones privadas.

**Verificación:** `show ip interface brief` → `GigabitEthernet0/0/0` (`100.100.100.1`) y
`GigabitEthernet0/0/1` (`200.10.10.1`) las dos up/up.

### Tarea 3 — PC-EXT: IP fija (GUI)

**Pasos:** doble clic en **PC-EXT** → **Desktop** → **IP Configuration** → **Static** →
IP `100.100.100.2` — Máscara `255.255.255.252` — Gateway `100.100.100.1`.

**Verificación:** Desktop → **Command Prompt** → `ping 100.100.100.1` → responde (Router ISP).

**Confirmado:** ping OK una vez armado el circuito PC-EXT → DSL-Modem-PT → Nube (mapeo DSL) →
Router ISP de la Tarea 1. **Tareas 1, 2 y 3 cerradas.**

### Tarea 4 — Router 4331: ruta por defecto hacia Router ISP (CLI)

**Objetivo:** que el 4331 sepa qué hacer con cualquier paquete cuyo destino no sea una de sus redes
conocidas (VLANs, DMZ) — mandarlo hacia Router ISP.
**Para qué:** sin esto, cuando el WEB-SERVER le conteste a PC-EXT, el 4331 no va a saber por dónde
sacar esa respuesta y la descarta.

**Pasos (CLI, doble clic en el Router 4331 → CLI):**
```
enable
configure terminal
ip route 0.0.0.0 0.0.0.0 200.10.10.1
end
write memory
```
- `ip route 0.0.0.0 0.0.0.0 200.10.10.1`: es la **ruta por defecto** — una regla "comodín" que dice
  "todo lo que no sepas a dónde mandar, mandalo para 200.10.10.1 (Router ISP)". Sin ella, el 4331
  solo conoce las redes conectadas directamente a sus propias interfaces.

**Verificación:** `show ip route` → debe aparecer una línea `S* 0.0.0.0/0 [1/0] via 200.10.10.1`.

**Confirmado:** `show ip route` en R-PERIMETRO muestra `Gateway of last resort is 200.10.10.1 to
network 0.0.0.0`. **Tarea 4 cerrada.**

### Tarea 5 — Router 4331: NAT estático del WEB-SERVER (CLI)

**Objetivo:** publicar `192.168.40.10` (privada, no alcanzable desde afuera) usando la IP pública
del propio 4331 (`200.10.10.2`), solo en los puertos de HTTPS y FTP.
**Para qué:** es el mecanismo que le permite a "Externos" llegar al server sin que la organización
tenga que exponer su red privada directamente a Internet — el punto 4 de la consigna, fila
"Externos".

**Pasos (CLI, doble clic en el Router 4331 → CLI):**
```
enable
configure terminal

! Marcar de qué lado de la casa está cada interfaz (obligatorio para que el NAT funcione)
interface GigabitEthernet0/0/1
 ip nat inside
exit

interface GigabitEthernet0/0/2
 ip nat outside
exit

! NAT estático: puerto 443 (HTTPS) y 20/21 (FTP) del propio 200.10.10.2 apuntan al WEB-SERVER
ip nat inside source static tcp 192.168.40.10 443 200.10.10.2 443
ip nat inside source static tcp 192.168.40.10 21 200.10.10.2 21
ip nat inside source static tcp 192.168.40.10 20 200.10.10.2 20

end
write memory
```
- `ip nat inside` / `ip nat outside`: cada interfaz del router tiene que quedar marcada de qué lado
  está — `inside` (red privada propia) o `outside` (hacia Internet). El NAT solo traduce
  direcciones cuando el tráfico cruza de un lado marcado al otro. Acá solo marcamos `dmz` (por
  donde se llega al WEB-SERVER) y `outside` — no hace falta marcar las VLANs 10/20/30 porque esta
  regla de NAT no las involucra (eso sería para que las PCs naveguen a Internet, que es opcional y
  no lo estamos haciendo).
- `ip nat inside source static tcp <IP privada> <puerto> <IP pública> <puerto>`: la IP pública va
  escrita a mano (`200.10.10.2`). *(Nota: la sintaxis real de Cisco IOS admite reemplazar la IP
  pública por `interface GigabitEthernet0/0/2` para que tome automáticamente la IP vigente de esa
  interfaz — pero el IOS simulado de Packet Tracer no la soporta, tira `Invalid input`. Por eso acá
  va la IP fija; si algún día cambiara la IP de `outside` habría que actualizar esta regla a mano.)*
- Puertos usados: `443` = HTTPS, `21` = FTP (canal de control), `20` = FTP (canal de datos, modo
  activo). Si al final probás FTP y falla solo la transferencia del archivo (el login sí entra), es
  la limitación de "sin inspección de protocolo" que ya vimos en `FIREWALL_LICENSE_ISSUE.md` — un
  router con ACLs no entiende los puertos dinámicos que negocia el modo pasivo de FTP. Si pasa,
  probá el cliente FTP en modo activo, o avisame y lo vemos juntos.

**Verificación:** `show ip nat translations` (después de generar tráfico desde PC-EXT en la
Verificación de cierre) → deben aparecer las traducciones `200.10.10.2:443 ↔ 192.168.40.10:443`, etc.

**Confirmado:** `show ip nat translations` muestra las 3 traducciones (`20`, `21`, `443`) apuntando
a `192.168.40.10`. **Tarea 5 cerrada.**

### Tarea 6 — Router 4331: ACL de entrada en `outside` (CLI)

**Objetivo:** que desde Internet **solo** se pueda llegar al WEB-SERVER publicado por HTTPS/FTP —
nada más, ni siquiera ping, ni otros puertos, ni otras redes.
**Para qué:** sin esta ACL, el NAT por sí solo no protege nada — cualquiera que sepa la IP pública
podría intentar cualquier puerto. Esto es lo mínimo de "mínimo privilegio" para Externos que ya
podemos aplicar ahora (la matriz completa de los otros segmentos es Fase 4).

**Pasos (CLI, doble clic en el Router 4331 → CLI):**
```
enable
configure terminal

ip access-list extended ACL-OUTSIDE-IN
 remark Externos: solo HTTPS y FTP hacia el WEB-SERVER publicado
 permit tcp any host 200.10.10.2 eq 443
 permit tcp any host 200.10.10.2 eq 21
 permit tcp any host 200.10.10.2 eq 20
 deny ip any any
exit

interface GigabitEthernet0/0/2
 ip access-group ACL-OUTSIDE-IN in
exit

end
write memory
```
- `ip access-list extended <NOMBRE>`: crea una ACL **nombrada** (en vez de numerada) — más fácil de
  leer y de editar después (se puede agregar una línea sin reescribir todo).
- `permit tcp any host 200.10.10.2 eq 443`: "dejá pasar tráfico TCP desde cualquier origen (`any`)
  hacia el host `200.10.10.2` (la IP pública), puerto 443". Importante: se compara contra la IP
  **pública** (`200.10.10.2`), no contra la privada del WEB-SERVER — porque esta ACL de entrada se
  evalúa **antes** de que el router traduzca la dirección (el NAT ocurre después, al decidir para
  dónde rutear).
- `deny ip any any` al final: por las dudas, aunque ya existe un "deny" implícito al final de toda
  ACL de Cisco — se lo deja explícito para que se vea clarito en `show access-lists` (ayuda para el
  informe).
- `ip access-group ACL-OUTSIDE-IN in`: aplica la ACL a la interfaz `outside`, en dirección
  **entrada** (`in` = lo que llega a esa boca desde afuera). Si se pusiera `out`, filtraría lo que
  *sale* por ahí, que no es lo que queremos.

**Verificación:** `show access-lists` → debe listar las 3 líneas `permit` + el `deny ip any any`.

**Confirmado:** `show access-lists` muestra las 3 `permit` (443, ftp/21, 20) + `deny ip any any`.
**Tarea 6 cerrada.**

### Tarea 7 — WEB-SERVER: activar HTTP/HTTPS/FTP (GUI)

**Objetivo:** que el WEB-SERVER realmente responda por esos servicios (si están apagados, da igual
cuán bien esté el NAT/ACL — no hay nada del otro lado que conteste).
**Para qué:** consigna punto 4 pide que Externos pueda "subir archivos" por FTP — hace falta un
usuario con permiso de escritura.

**Pasos:** doble clic en **WEB-SERVER** → pestaña **Services**:
1. **HTTP:** confirmá que el servicio esté **On** (y si aparece un toggle separado de HTTPS,
   activalo también).
2. **FTP:** activalo (**On**). Agregá un usuario nuevo, por ejemplo `externo` / `externo123`, con
   los permisos **Write** y **Read** tildados (sin Write no va a poder subir nada).

### Verificación de cierre de Fase 3

| Prueba | Desde | Cómo | Resultado esperado |
|---|---|---|---|
| Conectividad WAN externa | PC-EXT | `ping 100.100.100.1` | Responde (Router ISP) |
| Ruta hasta la IP pública | PC-EXT | `ping 200.10.10.2` | Puede fallar si el ICMP no está permitido en la ACL — **es esperado**, no es un error (la ACL de Externos no incluye ICMP) |
| HTTPS al server publicado | PC-EXT | Desktop → **Web Browser** → `https://200.10.10.2` | Carga la página del WEB-SERVER |
| FTP al server publicado | PC-EXT | Desktop → **Command Prompt** → `ftp 200.10.10.2`, login `externo`/`externo123`, `put <archivo>` | Sube el archivo OK |
| HTTP bloqueado (para el informe) | PC-EXT | Web Browser → `http://200.10.10.2` | Debe fallar / timeout — confirma que la ACL bloquea lo no permitido |
| Traducciones NAT | Router 4331 | `show ip nat translations` | Aparecen las sesiones de PC-EXT hacia `200.10.10.2` |

La prueba de "HTTP bloqueado" es justo el tipo de captura que pide la entrega (permitido vs.
bloqueado) — convendría sacarle screenshot a esa y a la de HTTPS/FTP exitosos para el informe final.

**Confirmado (2026-09-15):** Tareas 1-7 hechas y verificadas paso a paso durante la sesión (NAT y
ACL con salida real chequeada — ver Tareas 5 y 6). El usuario confirmó que el resto de la batería
de esta tabla también pasó; las capturas/logs puntuales de cada fila se van a tomar todos juntos
más adelante (para el informe final de Fase 5), no hace falta repetirlos ahora. **Fase 3 completa.**

---

## Fase 4 — Políticas de seguridad / ACLs con mínimo privilegio

**Objetivo de la fase** (`PLAN_FASES.md`): aplicar la matriz completa de ACLs por segmento
(Administración, Sistemas, Usuarios) contra el WEB-SERVER, con **mínimo privilegio** — cada
segmento solo puede usar los servicios que le corresponden, todo lo demás queda bloqueado.
**Estado al cierre:** la matriz permitido/bloqueado se comporta según la tabla de `PLAN_FASES.md`
(p.ej. HTTPS OK y HTTP bloqueado para Usuarios).

**Recordatorio de la matriz (`PLAN_FASES.md`):**

| Origen | Permitido (hacia WEB-SERVER salvo aclaración) | Denegado |
|---|---|---|
| Administración (VLAN10) | HTTPS, FTP | todo lo demás |
| Usuarios (VLAN30) | HTTPS | todo lo demás |
| Sistemas (VLAN20) | HTTPS, HTTP, FTP, SSH, ICMP **+ redes Usuarios y Admin** | todo lo demás |
| Externos (Internet) | HTTPS, FTP (ya resuelto en Fase 3, `ACL-OUTSIDE-IN`) | todo lo demás |

**Nota de alcance — decisión sobre SSH (riesgo abierto en `PLAN_FASES.md`):** el Server-PT de
Packet Tracer no ofrece servidor SSH real, así que la regla "Sistemas → SSH" no se puede probar
*funcionalmente* contra el WEB-SERVER. Se resuelve **a nivel ACL únicamente**: se agrega el
`permit` para el puerto 22 (queda la regla correcta y demostrable con `show access-lists`), sin
agregar hardware nuevo a la topología — no lo pide la consigna y ya hay una nota en la memoria del
proyecto para no sumar dispositivos fuera de lo pedido. Esto se va a aclarar en el informe final
(Fase 5) como limitación conocida del simulador, no de la configuración.

**Consecuencia esperada (a propósito):** después de esta fase, Administración y Usuarios **dejan
de poder pingear ni acceder a nada fuera de HTTPS/FTP contra el WEB-SERVER** — ni siquiera a su
propio gateway por ICMP, ni a las otras VLANs (el `ping` inter-VLAN que validamos en Fase 2 para
Admin/Usuarios se bloquea ahora a propósito). Solo Sistemas conserva ICMP y acceso completo a las
redes de Admin y Usuarios, tal cual pide la matriz. Para probar "permitido/bloqueado" en
Admin/Usuarios usá el navegador o FTP, no `ping` (el `ping` da bloqueado siempre para esos dos,
es lo esperado).

- [x] Tarea 1 — Router 4331: crear las 3 ACLs extendidas nombradas (CLI)
- [x] Tarea 2 — Router 4331: aplicar cada ACL en la subinterfaz correspondiente (CLI)
- [x] Verificación de cierre de Fase 4

### Tarea 1 — Router 4331: crear las 3 ACLs extendidas nombradas (CLI)

**Objetivo:** una ACL por VLAN interna, con las reglas de la matriz.
**Para qué:** cada ACL es la traducción directa de una fila de la matriz a reglas que el router
puede evaluar. Al ser **nombradas** (no numeradas) se pueden releer y editar fácil — mismo criterio
que se usó en Fase 3 con `ACL-OUTSIDE-IN`.

**Pasos (CLI, doble clic en el Router 4331 → CLI):**
```
enable
configure terminal

! ---- Administracion (VLAN10): solo HTTPS y FTP hacia el WEB-SERVER ----
ip access-list extended ACL-ADMIN-IN
 remark Administracion: solo HTTPS y FTP hacia el WEB-SERVER, resto denegado
 permit tcp 192.168.10.0 0.0.0.255 host 192.168.40.10 eq 443
 permit tcp 192.168.10.0 0.0.0.255 host 192.168.40.10 eq 21
 permit tcp 192.168.10.0 0.0.0.255 host 192.168.40.10 eq 20
 deny ip any any
exit

! ---- Usuarios (VLAN30): solo HTTPS hacia el WEB-SERVER ----
ip access-list extended ACL-USUARIOS-IN
 remark Usuarios: solo HTTPS hacia el WEB-SERVER, resto denegado
 permit tcp 192.168.30.0 0.0.0.255 host 192.168.40.10 eq 443
 deny ip any any
exit

! ---- Sistemas (VLAN20): HTTPS/HTTP/FTP/SSH/ICMP al WEB-SERVER + redes Admin y Usuarios ----
ip access-list extended ACL-SISTEMAS-IN
 remark Sistemas: HTTPS, HTTP, FTP, SSH e ICMP hacia el WEB-SERVER
 permit tcp 192.168.20.0 0.0.0.255 host 192.168.40.10 eq 443
 permit tcp 192.168.20.0 0.0.0.255 host 192.168.40.10 eq 80
 permit tcp 192.168.20.0 0.0.0.255 host 192.168.40.10 eq 21
 permit tcp 192.168.20.0 0.0.0.255 host 192.168.40.10 eq 20
 permit tcp 192.168.20.0 0.0.0.255 host 192.168.40.10 eq 22
 permit icmp 192.168.20.0 0.0.0.255 host 192.168.40.10
 remark Sistemas: acceso completo a las redes de Administracion y Usuarios
 permit ip 192.168.20.0 0.0.0.255 192.168.10.0 0.0.0.255
 permit ip 192.168.20.0 0.0.0.255 192.168.30.0 0.0.0.255
 deny ip any any
exit

end
write memory
```
- `192.168.10.0 0.0.0.255`: red de origen + **máscara wildcard** (ver `GLOSARIO.md`) — significa
  "cualquier host de la red 192.168.10.0/24", no una IP puntual.
- `host 192.168.40.10`: destino puntual, siempre el WEB-SERVER (única IP de la DMZ).
- Puertos: `443` HTTPS, `80` HTTP, `21`/`20` FTP (control/datos, mismo criterio que Fase 3), `22`
  SSH (ver nota de alcance arriba). `permit icmp ...` sin `eq` porque ICMP no usa puertos.
- `permit ip 192.168.20.0 0.0.0.255 192.168.10.0 0.0.0.255`: `ip` (no `tcp`) permite **cualquier
  protocolo** entre esas dos redes — es "acceso completo", no limitado a un servicio, tal cual pide
  la fila de Sistemas en la matriz.
- `deny ip any any` al final de cada ACL: explícito por claridad (ya existe implícito), mismo
  criterio que Fase 3.
- **Todavía no se aplican a ninguna interfaz** — eso es la Tarea 2. Crearlas primero y aplicarlas
  después evita dejar una VLAN bloqueada a mitad de una edición.

**Verificación:** `show access-lists` → deben figurar las 3 ACLs (`ACL-ADMIN-IN`, `ACL-USUARIOS-IN`,
`ACL-SISTEMAS-IN`) con sus reglas, más la `ACL-OUTSIDE-IN` de Fase 3.

### Tarea 2 — Router 4331: aplicar cada ACL en la subinterfaz correspondiente (CLI)

**Objetivo:** activar el filtrado, aplicando cada ACL en sentido **entrada** sobre la subinterfaz
por la que llega el tráfico de esa VLAN.
**Para qué:** una ACL creada pero no aplicada a ninguna interfaz no filtra nada — es solo una lista
guardada. `in` en la subinterfaz de cada VLAN filtra lo que esa VLAN manda **hacia** el router,
antes de que se rutee a cualquier otro lado (mismo criterio que `ACL-OUTSIDE-IN` en Fase 3).

**Pasos (CLI, doble clic en el Router 4331 → CLI):**
```
enable
configure terminal

interface GigabitEthernet0/0/0.10
 ip access-group ACL-ADMIN-IN in
exit

interface GigabitEthernet0/0/0.20
 ip access-group ACL-SISTEMAS-IN in
exit

interface GigabitEthernet0/0/0.30
 ip access-group ACL-USUARIOS-IN in
exit

end
write memory
```

**Verificación:** `show ip interface GigabitEthernet0/0/0.10` (y `.20`/`.30`) → debe listar la ACL
correspondiente aplicada como "inbound".

**Troubleshooting real (apareció en ejecución):** con las ACLs de la Tarea 1 tal cual, el `ping`
de PC-SIS1 al WEB-SERVER andaba bien, pero PC-SIS1 → PC-ADM1 y PC-SIS1 → PC-USR1 daban *timeout*.
Causa: el tráfico WEB-SERVER↔VLANs solo cruza **una** subinterfaz con ACL (la del cliente; el lado
`dmz`, `Gi0/0/1`, no tiene ACL aplicada), pero el tráfico Sistemas↔Admin/Usuarios cruza **dos**
subinterfaces con ACL — la ida (permitida por `ACL-SISTEMAS-IN`) y la vuelta, que entra al router
por `Gi0/0/0.10` o `.30` y ahí `ACL-ADMIN-IN`/`ACL-USUARIOS-IN` no tenían ninguna regla que dejara
pasar una respuesta hacia Sistemas (solo tenían reglas hacia el WEB-SERVER). Con una ACL sin
estado, permitir la ida de un lado no alcanza para que vuelva la respuesta del otro lado.

**Solución aplicada:** agregar en `ACL-ADMIN-IN` y `ACL-USUARIOS-IN` una regla de **solo vuelta**
hacia la red de Sistemas, usando `established` (TCP) y el tipo de mensaje `echo-reply` (ICMP) — ver
`GLOSARIO.md`. Deja pasar la respuesta sin darle a Administración/Usuarios permiso para *iniciar*
tráfico hacia Sistemas (verificado: `ping` de PC-ADM1 a PC-SIS1 sigue bloqueado, como corresponde).

```
enable
configure terminal

ip access-list extended ACL-ADMIN-IN
 no deny ip any any
 permit tcp 192.168.10.0 0.0.0.255 192.168.20.0 0.0.0.255 established
 permit icmp 192.168.10.0 0.0.0.255 192.168.20.0 0.0.0.255 echo-reply
 deny ip any any
exit

ip access-list extended ACL-USUARIOS-IN
 no deny ip any any
 permit tcp 192.168.30.0 0.0.0.255 192.168.20.0 0.0.0.255 established
 permit icmp 192.168.30.0 0.0.0.255 192.168.20.0 0.0.0.255 echo-reply
 deny ip any any
exit

end
write memory
```
- `no deny ip any any` antes de agregar las reglas nuevas: en una ACL nombrada, las líneas nuevas
  se agregan **al final** si no se les da un número de secuencia — agregarlas después del `deny ip
  any any` las dejaría muertas (nunca se evaluarían). Por eso se saca el `deny`, se agregan las
  reglas de vuelta, y se lo vuelve a poner al final.

**Confirmado:** con este agregado, PC-SIS1 → `ping 192.168.10.10` y `ping 192.168.30.10` responden
(0% loss); PC-ADM1 → `ping 192.168.20.10` sigue bloqueado. **Tarea 2 cerrada.**

### Verificación de cierre de Fase 4

| Prueba | Desde | Cómo | Resultado esperado |
|---|---|---|---|
| HTTPS al WEB-SERVER | PC-ADM1 | Web Browser → `https://192.168.40.10` | Carga la página |
| HTTP al WEB-SERVER | PC-ADM1 | Web Browser → `http://192.168.40.10` | Bloqueado / timeout |
| FTP al WEB-SERVER | PC-ADM1 | Command Prompt → `ftp 192.168.40.10`, login, `put` | Sube el archivo OK |
| Ping al WEB-SERVER | PC-ADM1 | `ping 192.168.40.10` | Bloqueado (ICMP no está en la matriz de Admin) |
| HTTPS al WEB-SERVER | PC-USR1 | Web Browser → `https://192.168.40.10` | Carga la página |
| HTTP al WEB-SERVER | PC-USR1 | Web Browser → `http://192.168.40.10` | Bloqueado / timeout |
| FTP al WEB-SERVER | PC-USR1 | Command Prompt → `ftp 192.168.40.10` | Bloqueado, no conecta |
| HTTPS/HTTP/FTP al WEB-SERVER | PC-SIS1 | Browser + `ftp 192.168.40.10` | Los tres OK |
| Ping al WEB-SERVER | PC-SIS1 | `ping 192.168.40.10` | Responde |
| Acceso a red Admin | PC-SIS1 | `ping 192.168.10.10` (PC-ADM1) | Responde |
| Acceso a red Usuarios | PC-SIS1 | `ping 192.168.30.10` (PC-USR1) | Responde |
| Admin no llega a Sistemas | PC-ADM1 | `ping 192.168.20.10` (PC-SIS1) | Bloqueado (a propósito) |
| Reglas SSH cargadas | Router 4331 | `show access-lists ACL-SISTEMAS-IN` | Aparece el `permit tcp ... eq 22` (no se prueba funcionalmente, ver nota de alcance) |
| Contadores de hits | Router 4331 | `show access-lists` | Los `permit`/`deny` muestran matches (`(N matches)`) después de generar tráfico |

Cuando corras esta batería, pasame resultados (capturas o texto) y lo valido contra este checklist
antes de dar la Fase 4 por cerrada.

**Confirmado (2026-09-16):** las 15 pruebas de la batería pasaron. Administración y Usuarios
acceden al WEB-SERVER solo por los servicios de su fila de la matriz (resto bloqueado, incluido
ICMP); Sistemas accede al WEB-SERVER por los 5 servicios y a las redes de Administración y Usuarios
(ida y vuelta); Administración no puede iniciar tráfico hacia Sistemas (verificado). El permiso SSH
de Sistemas quedó configurado y confirmado por `show access-lists`, sin prueba funcional por la
limitación del Server-PT (ver nota de alcance al inicio de la fase). **Fase 4 completa.**
