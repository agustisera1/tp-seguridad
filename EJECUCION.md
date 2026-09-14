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

---

## Fase 2 — Inter-VLAN y conectividad L3 interna

**Objetivo de la fase** (`PLAN_FASES.md`): que las 3 VLANs internas (Admin/Sistemas/Usuarios) se
puedan comunicar entre sí y alcancen la DMZ y la ASA.

**Pivot confirmado (no es más un "checkpoint", ya se decidió):** la idea original era que la ASA
hiciera de gateway de las 3 VLANs con subinterfaces 802.1Q sobre un trunk. En Fase 1 ya vimos que
esta ASA corre con **licencia Base**: tope de ~2 `nameif` completas + 1 restringida, **sin**
soporte de trunk/subinterfaces (eso es una función de Security Plus). Con `outside` + `dmz` ya
ocupando las 2 completas, no hay margen para 3 VLANs más en la ASA.

**Solución:** un switch de capa 3 (**SW-L3**, modelo 3560) pasa a hacer el ruteo entre VLANs — sus
SVIs (`interface Vlan10/20/30`) son ahora el gateway de cada segmento (mismas IPs `.1` que ya
estaban planificadas, así que **no hay que tocar las PCs**). La ASA se queda solo con perímetro +
DMZ, y se conecta a SW-L3 con un 3er `nameif` llamado `inside`, pero como **enlace routeado punto
a punto** (sin trunk) sobre una red de tránsito nueva `192.168.50.0/30`.

**Riesgo a validar en esta fase:** la licencia Base permite 3 VLANs (`nameif`) en total, pero la
3ra viene con la restricción de no poder iniciar tráfico hacia **dos** interfaces distintas a la
vez — solo hacia una. `inside` necesita hablar sí o sí con `dmz` (las VLANs acceden al
WEB-SERVER, es el corazón de la matriz de ACLs) pero no necesita iniciar tráfico hacia `outside`
(la navegación interna→Internet es **opcional** según el plan). Por eso se aplica
`no forward interface outside` en `inside`: bloquea que `inside` inicie tráfico hacia `outside`,
pero deja libre `inside`↔`dmz`. Si igual tira error de licencia, lo documentamos acá y decidimos
alternativa (por ejemplo, restringir al revés, o repensar cuál interface queda como la "libre").

- [ ] Tarea 1 — Agregar SW-L3 y recablear
- [ ] Tarea 2 — SW-L3: VLANs, SVIs (gateways), trunk hacia SW-LAN, ruteo
- [ ] Tarea 3 — SW-LAN: confirmar el trunk (ahora apunta a SW-L3)
- [ ] Tarea 4 — ASA 5505: interfaz `inside` (enlace routeado hacia SW-L3)
- [ ] Tarea 5 — Rutas estáticas (SW-L3 → ASA, ASA → VLANs internas)
- [ ] Verificación de cierre de Fase 2

### Tarea 1 — Agregar SW-L3 y recablear

**Objetivo:** sumar el switch que va a hacer inter-VLAN y acomodar el cableado: el trunk que
armaste en Fase 1 (SW-LAN → ASA) pasa a ir de SW-LAN → SW-L3, y agregás un cable nuevo
SW-L3 → ASA.

**Para qué:** es el cambio físico que habilita el pivot — sin este paso, los comandos de las
tareas siguientes no tienen dónde aplicarse.

**Pasos (GUI):**
1. Del panel de dispositivos (abajo a la izquierda), categoría **Switches**, arrastrá al lienzo un
   **3560** (soporta rutear, a diferencia del 2960 de SW-LAN/SW-DMZ). Nombralo mentalmente "SW-L3"
   (el hostname real se lo ponemos por CLI en la Tarea 2).
2. Ubicalo entre SW-LAN y la ASA en el lienzo (no importa la posición exacta, solo prolijidad).
3. **Desconectá** el cable que hoy va de SW-LAN (el puerto trunk que configuraste en Fase 1
   Tarea 5, ej. `GigabitEthernet0/1`) a la ASA. Click en el cable → aparece opción de eliminarlo,
   o simplemente click en el conector **Delete** (tijera) de la barra de herramientas y click en
   el cable.
4. Conectá un cable **Copper Straight-Through** desde ese mismo puerto de SW-LAN hasta un puerto
   de SW-L3 (ej. `GigabitEthernet0/1`).
5. Conectá otro cable **Copper Straight-Through** desde otro puerto de SW-L3 (ej.
   `GigabitEthernet0/2`) hasta el puerto de la ASA que había quedado libre desde Fase 1 (el que en
   su momento pensábamos usar para el trunk directo, ej. `Ethernet0/2`).

**Verificación:** los tres cables (SW-LAN↔SW-L3, SW-L3↔ASA, y los que ya existían) deberían verse
con puntos verdes o ámbar parpadeante (todavía no van a estar 100% up hasta que configures los
puertos en las tareas siguientes — es normal).

### Tarea 2 — SW-L3: VLANs, SVIs (gateways), trunk hacia SW-LAN, ruteo

**Objetivo:** crear las VLANs 10/20/30 en SW-L3 (tienen que existir localmente para que las SVIs
funcionen), darles IP a las SVIs (van a ser el gateway de cada VLAN), habilitar el trunk hacia
SW-LAN, prender el ruteo entre VLANs, y dejar el puerto hacia la ASA como **routeado** (no
switchport) con la IP de tránsito.

**Para qué:** las SVIs (`interface Vlan10`, etc.) son interfaces virtuales que le dan IP a una
VLAN dentro del switch — es lo que convierte a un switch normal en "capa 3". `ip routing` es el
comando que prende el ruteo entre esas VLANs (sin él, el switch solo haría switching, no rutearía
entre redes distintas aunque tengan SVIs). El puerto hacia la ASA se pone `no switchport` porque
ahí no va tráfico de switch (VLANs), va un enlace ruteado punto a punto como si fuera un router.

**Pasos (CLI, doble clic en SW-L3 → CLI):**
```
enable
configure terminal
hostname SW-L3
ip routing

vlan 10
 name ADMINISTRACION
exit
vlan 20
 name SISTEMAS
exit
vlan 30
 name USUARIOS
exit

! Gateway de Administración
interface Vlan10
 ip address 192.168.10.1 255.255.255.0
 no shutdown
exit

! Gateway de Sistemas
interface Vlan20
 ip address 192.168.20.1 255.255.255.0
 no shutdown
exit

! Gateway de Usuarios
interface Vlan30
 ip address 192.168.30.1 255.255.255.0
 no shutdown
exit

! Reemplazá GigabitEthernet0/1 por el puerto real hacia SW-LAN
interface GigabitEthernet0/1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30
 no shutdown
 description Trunk hacia SW-LAN
exit

! Reemplazá GigabitEthernet0/2 por el puerto real hacia la ASA — enlace routeado, NO switchport
interface GigabitEthernet0/2
 no switchport
 ip address 192.168.50.2 255.255.255.252
 no shutdown
 description Enlace routeado hacia ASA (inside)
exit

end
write memory
```
- Si `switchport trunk encapsulation dot1q` tira error en el 3560 (algunos modelos ya vienen fijos
  en dot1q), seguí directo con `switchport mode trunk`, no es un problema — misma nota que en
  Fase 1.
- `no switchport`: convierte un puerto físico de "puerto de switch" a "puerto routeado" — recién
  ahí acepta una IP directamente, como si fuera la interfaz de un router.

**Verificación:**
- `show vlan brief` → VLANs 10/20/30 creadas.
- `show ip interface brief` → `Vlan10/20/30` y `GigabitEthernet0/2` deben figurar `up/up` con sus IPs.
- `show interfaces trunk` → el puerto hacia SW-LAN en modo trunk, VLANs 10,20,30 permitidas.

### Tarea 3 — SW-LAN: confirmar el trunk (ahora apunta a SW-L3)

**Objetivo:** no hay cambios de configuración — el trunk que ya armaste en Fase 1 Tarea 5 sigue
sirviendo tal cual, porque solo cambió el cable de destino (ahora va a SW-L3 en vez de a la ASA).
Esta tarea es solo de **verificación**.

**Para qué:** confirmar que el recableado de la Tarea 1 no rompió nada y que el trunk sigue activo
contra el nuevo vecino.

**Pasos:** ninguno (no hace falta tocar CLI en SW-LAN).

**Verificación (CLI, doble clic en SW-LAN → CLI):**
- `show interfaces trunk` → el puerto hacia SW-L3 (mismo puerto de siempre) debe seguir en modo
  trunk con VLANs 10,20,30 permitidas.
- `show vlan brief` → los puertos de las PCs siguen en sus VLANs de Fase 1.

### Tarea 4 — ASA 5505: interfaz `inside` (enlace routeado hacia SW-L3)

**Objetivo:** configurar el 3er `nameif` de la ASA, `inside`, como enlace punto a punto hacia
SW-L3 (sin trunk), con la restricción de licencia aplicada preventivamente.

**Para qué:** es el único punto de contacto entre las VLANs internas y el resto de la red (DMZ y
outside) — todo el tráfico inter-segmento pasa filtrado por acá (más adelante, en Fase 4, con
ACLs de mínimo privilegio).

**Pasos (CLI, doble clic en ASA → CLI):**
```
enable
configure terminal

! ---- inside: hacia SW-L3 (enlace routeado, sin trunk) ----
interface Vlan1
 nameif inside
 security-level 100
 ip address 192.168.50.1 255.255.255.252
 no forward interface outside
 no shutdown
exit

! Reemplazá Ethernet0/2 por el puerto real conectado a SW-L3
interface Ethernet0/2
 switchport access vlan 1
 no shutdown
exit

end
write memory
```
- `security-level 100`: la interfaz más confiable (la LAN interna), igual que en el diseño
  original.
- `no forward interface outside`: es la mitigación de licencia explicada arriba — `inside` puede
  iniciar tráfico hacia `dmz` (necesario) pero no hacia `outside` (opcional). Si al pegar este
  comando la ASA tira error de licencia igual, avisame con el mensaje exacto y lo resolvemos acá
  antes de seguir.
- `interface Vlan1`: es la VLAN que liberamos en Fase 1 Tarea 4 (`no nameif` / `no ip address`) —
  ahora la reutilizamos para `inside`, pero como enlace simple (no trunk).

**Verificación:**
- `show interface ip brief` → `inside` debe figurar `up` con `192.168.50.1`.
- Desde la ASA: `ping inside 192.168.50.2` → debe responder (SW-L3, una vez tenga su lado
  configurado en la Tarea 2).

### Tarea 5 — Rutas estáticas (SW-L3 → ASA, ASA → VLANs internas)

**Objetivo:** decirle a cada lado del enlace de tránsito cómo llegar a las redes que están "del
otro lado" y que no conoce directamente.

**Para qué:** tener las interfaces con IP no alcanza — sin rutas, SW-L3 no sabe cómo llegar a la
DMZ/outside, y la ASA no sabe cómo llegar a las VLANs 10/20/30 (que están un salto más allá de
SW-L3, no conectadas directo a la ASA).

**Pasos — en SW-L3 (CLI):**
```
enable
configure terminal
! Todo lo que no sea 10/20/30 (o sea, DMZ, outside, Internet) se manda a la ASA
ip route 0.0.0.0 0.0.0.0 192.168.50.1
end
write memory
```

**Pasos — en la ASA (CLI):**
```
enable
configure terminal
route inside 192.168.10.0 255.255.255.0 192.168.50.2
route inside 192.168.20.0 255.255.255.0 192.168.50.2
route inside 192.168.30.0 255.255.255.0 192.168.50.2
end
write memory
```
- En SW-L3 alcanza con una ruta por defecto (`0.0.0.0/0`) porque todo lo que no es una VLAN local
  vive "para el lado de la ASA".
- En la ASA hace falta una ruta por cada red VLAN porque no hay una ruta por defecto genérica hacia
  `inside` (la ruta por defecto de la ASA, si existe, va a apuntar hacia `outside`/Internet, no
  hacia adentro).

**Verificación:**
- SW-L3: `show ip route` → debe aparecer `S* 0.0.0.0/0 [1/0] via 192.168.50.1`.
- ASA: `show route` → deben aparecer las 3 rutas estáticas hacia `.10.0/.20.0/.30.0` vía `inside`.

### Verificación de cierre de Fase 2

| Prueba | Desde | Comando | Resultado esperado |
|---|---|---|---|
| PC a su gateway | PC-ADM1 (y SIS1/USR1) | `ping 192.168.10.1` (o `.20.1`/`.30.1`) | Responde (SVI en SW-L3) |
| Inter-VLAN | PC-ADM1 | `ping 192.168.20.10` (PC-SIS1) | Responde |
| VLAN a DMZ | PC-ADM1 (o cualquiera) | `ping 192.168.40.10` (WEB-SERVER) | Responde |
| SW-L3 a ASA inside | SW-L3 | `ping 192.168.50.1` | Responde |
| ASA inside a SW-L3 | ASA | `ping inside 192.168.50.2` | Responde |
| Rutas en SW-L3 | SW-L3 | `show ip route` | Default route vía `.50.1` |
| Rutas en ASA | ASA | `show route` | 3 rutas estáticas vía `.50.2` |

Cuando tengas esto corrido, pasame los resultados (capturas o texto de los comandos) y lo valido
antes de pasar a Fase 3.
