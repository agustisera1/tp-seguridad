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

*(se completa cuando arranquemos esta fase)*
