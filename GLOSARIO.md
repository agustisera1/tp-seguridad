# Glosario de orientación — TP1 Seguridad

Material de apoyo para armar y consultar mientras cableás en Packet Tracer. Todo está explicado
en 1–2 líneas y orientado a **qué es y para qué lo tocás en este TP**. La topología objetivo y
las direcciones están en `PLAN_FASES.md`.

---

## 1. Cómo moverte en Packet Tracer (lo básico de la GUI)

- **Área de trabajo (Logical):** el lienzo donde arrastrás dispositivos y los cableás. La pestaña
  **Physical** (arriba) es la vista de racks/edificios; para este TP usás **Logical**.
- **Panel inferior izquierdo:** el catálogo de dispositivos. Primero elegís una **categoría**
  (routers, switches, dispositivos de red, terminales…) y después el **modelo** concreto.
  Se arrastra al lienzo.
- **Agregar un dispositivo:** clic en su ícono del catálogo → clic (o arrastrar) en el lienzo.
- **Cablear:** elegís la categoría **Connections** (ícono del rayo) → elegís el tipo de cable →
  clic en el primer dispositivo (te pide el puerto) → clic en el segundo (te pide el puerto).
- **Configurar un dispositivo:** doble clic sobre él. Se abre una ventana con pestañas:
  - **Config:** configuración asistida por menús (IP, gateway, etc.) — la más fácil para PCs.
  - **CLI:** la consola de texto (para routers/switches/ASA) donde pegás comandos.
  - **Desktop** (en PCs): tenés **IP Configuration**, **Web Browser**, **Command Prompt** (para
    `ping`), etc.
  - **Services** (en el Server): activás/desactivás HTTP, HTTPS, FTP, DNS…
- **Guardar:** `Ctrl+S`. El archivo queda como `.pkt`.
- **Probar conectividad rápido:** el sobre/rayo del panel derecho (Add Simple PDU) manda un ping
  visual entre dos equipos; el estado sale abajo a la derecha.

---

## 2. Componentes (para qué sirve cada uno en este TP)

| Componente | Qué es | Para qué en el TP |
|---|---|---|
| **Router ISP** | Router que simula al proveedor de Internet | Borde WAN; enlaza la nube Internet con el equipo perimetral (red 200.10.10.0/30) |
| **Router 4331** | Router de Cisco; en este proyecto hace de "firewall" | El corazón de seguridad: separa outside (Internet) / VLANs internas / dmz, y aplica las ACLs. *(El plan original usaba una ASA 5505 — se cambió por un límite de licencia, ver `FIREWALL_LICENSE_ISSUE.md`)* |
| **Switch 2960** | Switch de capa 2 (24 bocas) | SW-LAN reparte las VLANs internas; SW-DMZ conecta el servidor |
| **Server-PT** | Servidor genérico | El WEB-SERVER: ofrece HTTP/HTTPS/FTP a probar |
| **PC** | Computadora cliente | Un host por segmento (Admin, Sistemas, Usuarios) + uno "Externo" en Internet |
| **Nube (Cloud-PT)** | Representa Internet | Extremo remoto; de ahí "vienen" los Externos. **Ojo:** sus puertos Ethernet no se puentean entre sí directamente — ver "Mapeo de puertos en la Nube" más abajo |
| **DSL-Modem-PT** | Modem DSL simulado | Dispositivo intermedio para colgar a PC-EXT de la Nube (ver Fase 3, Tarea 1): un lado Ethernet hacia el PC, un lado "línea" hacia la Nube |

> **Nota sobre nombres:** en la imagen los equipos se llaman PC-ADM1, PC-SIS1, PC-USR1,
> SW-LAN, SW-DMZ, WEB-SERVER. Conviene renombrarlos igual (doble clic → pestaña Config → Display Name).

---

## 3. Cables: cuál usar entre qué

En Packet Tracer, si no querés pensarlo, existe el cable **Automático** (ícono del rayo dorado),
que elige solo. Para hacerlo bien a mano:

| Entre… | Cable | Por qué |
|---|---|---|
| PC ↔ Switch | Cobre **directo** (*Copper Straight-Through*) | Dispositivos distintos |
| Switch ↔ Router / ASA | Cobre **directo** | Dispositivos distintos |
| Server ↔ Switch | Cobre **directo** | Dispositivos distintos |
| Router ↔ Router (o PC ↔ PC) | Cobre **cruzado** (*Copper Cross-Over*) | Dispositivos iguales |
| Router ISP ↔ Nube | Directo (o serial, según cómo modeles Internet) | Se define en Fase 3 |

> Regla mnemotécnica: **iguales = cruzado, distintos = directo**. (Los equipos modernos
> autonegocian, pero en el TP conviene respetarlo para que quede prolijo.)

---

## 4. Puertos / interfaces (cómo se llaman)

- **Interfaz:** una "puerta" del equipo — cada una se conecta a una red distinta. Un router
  necesita una interfaz por cada red a la que le da entrada/salida.
- **FastEthernet (Fa0/1):** boca de 100 Mbps. Los switches 2960 tienen Fa0/1…Fa0/24.
- **GigabitEthernet (Gi0/0/1):** boca de 1 Gbps. En el Router 4331 se llaman `GigabitEthernet0/0/0`,
  `0/0/1`, `0/0/2` (tres números porque el router organiza los puertos por slot/bahía, no es un
  capricho). En los switches 2960 son `Gi0/1` y `Gi0/2`, y suelen usarse para el *uplink* / trunk.
- **SFP / módulo GLC-T:** algunas bocas Gigabit del router vienen "vacías" (son solo el hueco, sin
  el conector de cobre soldado) y hay que ponerles un **módulo transceiver** (una pastillita que se
  inserta, como una memoria RAM chiquita) para activarlas. `GLC-T` es el módulo que convierte esa
  boca vacía en un puerto de cobre normal (cable de red común). Se pone con el equipo apagado.
- **NIM (Network Interface Module):** una tarjeta que se le agrega a un router modular para sumar
  puertos (Ethernet, seriales, etc.) que no vienen de fábrica. No se usó en este proyecto — alcanzó
  con activar la boca SFP con un GLC-T (ver `FIREWALL_LICENSE_ISSUE.md`).
- **Subinterfaz (Gi0/0/2.10):** una interfaz "virtual" sobre una física, atada a una VLAN. La usa
  el router para ser gateway de cada VLAN por el mismo cable físico (un trunk).
- *(Referencia histórica)* **Ethernet0/x (ASA):** así se llamaban las bocas en el diseño original
  con ASA 5505 (`Ethernet0/0` = outside, `Ethernet0/1` = inside). Ya no se usa esa numeración.

---

## 5. Conceptos de red (los que vas a tocar)

- **VLAN:** una "red lógica" dentro de un switch. Aísla el tráfico: la VLAN 10 (Admin) no ve a la
  VLAN 30 (Usuarios) salvo que algo las rutee. En el TP: VLAN 10/20/30.
- **Trunk (802.1Q):** un enlace que transporta **varias VLANs a la vez** (etiquetando cada trama
  con su número de VLAN). El cable SW-LAN ↔ ASA es un trunk.
- **Puerto de acceso (access):** boca que pertenece a **una sola** VLAN. Ahí se conectan las PCs.
- **Gateway (puerta de enlace):** la IP a la que un host manda todo lo que va "afuera" de su red.
  Para PC-ADM1 es 192.168.10.1 (la ASA).
- **Máscara / CIDR (/24, /30):** define cuántas IPs entran en la red. `/24` = 254 hosts;
  `/30` = 2 hosts (ideal para enlaces punto a punto como la WAN).
- **Inter-VLAN routing:** hacer que VLANs distintas se comuniquen. Lo hace un dispositivo de capa 3
  (en el TP, el Router 4331 con sus subinterfaces).
- **Router-on-a-stick:** el nombre que se le da a este truco de usar **un solo cable físico** (un
  trunk) y **subinterfaces** para que un router sea gateway de varias VLANs a la vez, en vez de
  necesitar un cable por VLAN. "Stick" = el único palito/cable que sube desde el switch.
- **ACL (Access Control List):** lista de reglas *permitir/denegar* tráfico según origen, destino
  y servicio. Es el núcleo de la consigna 4.
- **Máscara wildcard (wildcard mask):** la máscara "invertida" que usan las ACLs de Cisco para
  indicar un rango de IPs de origen/destino, en vez de la máscara de subred normal. Donde la
  máscara normal tiene un bit en `1` (esa parte tiene que coincidir exacto), la wildcard tiene `0`;
  donde la máscara normal tiene `0` (cualquier valor sirve), la wildcard tiene `255`. Para la red
  `192.168.10.0/24` (máscara `255.255.255.0`) la wildcard es `0.0.0.255`. Ejemplo de este TP:
  `permit tcp 192.168.10.0 0.0.0.255 host 192.168.40.10 eq 443` = "cualquier host de la red
  192.168.10.0/24 (Administración) hacia el WEB-SERVER, puerto 443". Es distinto del `host <IP>`
  (una sola IP exacta) y de `any` (cualquier IP), que ya se usaron en la ACL de Fase 3.
- **Mínimo privilegio:** dar solo los permisos estrictamente necesarios; todo lo no permitido, se
  niega. Las ACL terminan con un "deny" implícito que ayuda a esto.
- **DMZ (zona desmilitarizada):** red intermedia donde se ponen los servidores accesibles desde
  afuera (el WEB-SERVER), separada de la LAN interna para que un ataque al server no toque la LAN.
- **NAT / PAT:** traducción de direcciones. **NAT estático** publica el server privado
  (192.168.40.10) con una IP pública para que los Externos lo alcancen. **PAT** es el NAT "de
  muchos a uno" que usan las PCs para salir a Internet.
- **`ip nat inside` / `ip nat outside`:** en cada interfaz del router hay que marcar de qué lado
  está: `inside` (red privada propia) o `outside` (hacia Internet). El NAT solo traduce direcciones
  cuando el tráfico cruza de un lado marcado al otro — sin esto, `ip nat inside source static...`
  no hace nada.
- **Ruta por defecto (*default route*):** una regla de ruteo "comodín" (`ip route 0.0.0.0 0.0.0.0
  <siguiente-salto>`) que le dice al router "todo lo que no sepas a dónde mandar, mandalo para
  acá". Evita tener que escribir una ruta por cada red posible de Internet.
- **Puerto (TCP/UDP):** un número que identifica *qué servicio* va dentro de un paquete —
  443 = HTTPS, 80 = HTTP, 21 = FTP (control), 20 = FTP (datos), 22 = SSH. Las ACL lo usan
  (`eq 443`) para filtrar por servicio además de por IP.
- **Mapeo de puertos en la Nube (Cloud-PT):** dentro de la Nube, `Config → Connections` (Frame
  Relay / DSL / Cable) sirve para *traducir* un puerto de tecnología WAN (Modem, Coaxial, Serial)
  hacia un puerto Ethernet — nunca para unir dos puertos Ethernet entre sí directamente. Por eso
  para colgar a PC-EXT de la Nube hizo falta un **DSL-Modem-PT** de por medio (PC-EXT → modem →
  puerto `Modem4` de la Nube, con el mapeo `Modem4 <-> Ethernet6` agregado en la pestaña DSL) en
  vez de un segundo puerto Ethernet suelto. Detalle completo en `EJECUCION.md` (Fase 3, Tarea 1).
- **Dirección de una ACL (`in` / `out`) en una interfaz:** al aplicar una ACL con
  `ip access-group NOMBRE in` (o `out`) hay que decir si filtra lo que *entra* por esa boca o lo
  que *sale*. Una ACL de entrada en `outside` filtra lo que llega desde Internet antes de que el
  router decida a dónde rutearlo (y antes de que el NAT traduzca la dirección).
- *(Referencia histórica)* **security-level (ASA):** número 0–100 por interfaz que tenía el diseño
  original con ASA. Más alto = más confiable; por defecto el tráfico va de mayor a menor nivel. El
  Router 4331 **no tiene esto** — no hay bloqueo automático entre interfaces, todo se permite salvo
  que una ACL lo prohíba explícitamente (ver "ACL" arriba y el tradeoff en `FIREWALL_LICENSE_ISSUE.md`).
- **Licencia (en un equipo Cisco):** como un plan de celular — el fabricante vende el mismo equipo
  con distintos "paquetes" de funciones habilitadas según cuánto se paga. La ASA 5505 de este
  proyecto viene con la licencia más básica ("Base"), que limita cuántas interfaces se pueden usar
  a la vez. Fue la causa de tener que migrar a un router (detalle completo en
  `FIREWALL_LICENSE_ISSUE.md`).
- **`nameif` (comando de la ASA):** el comando que le pone nombre a una interfaz de la ASA
  (`outside`, `dmz`, etc.) y la activa para que empiece a pasar tráfico. Sin nombre, la interfaz no
  sirve aunque tenga IP configurada. La licencia Base limita cuántas interfaces se pueden "nombrar"
  a la vez — de ahí el problema documentado en `FIREWALL_LICENSE_ISSUE.md`. El Router 4331 no usa
  este comando: en un router, apenas le ponés una IP a una interfaz y la prendés (`no shutdown`),
  ya funciona.
- **`established` (palabra clave de ACL, solo TCP):** en una regla `permit tcp ... established`,
  el router deja pasar el paquete solo si tiene los bits `ACK` o `RST` prendidos — es decir, solo si
  es **parte de una conexión que ya empezó del otro lado**, nunca el primer paquete (`SYN`) de una
  conexión nueva. Es el truco para que una ACL sin estado deje volver la respuesta de un tráfico
  permitido en la otra ACL, sin abrir la posibilidad de iniciar conexiones nuevas en ese sentido.
  Se usó en Fase 4 para que las respuestas de Administración/Usuarios lleguen de vuelta a Sistemas
  sin darles a Administración/Usuarios permiso para iniciar tráfico hacia Sistemas.
- **Tipo de mensaje ICMP (`echo-reply` vs `echo-request`):** un `ping` en realidad son dos mensajes
  ICMP distintos: el que sale (`echo-request`) y el que contesta (`echo-reply`). Una ACL puede
  permitir uno sin el otro — `permit icmp <red> <red> echo-reply` deja volver solo la *respuesta*
  de un ping ajeno, sin permitir que esa red *inicie* un ping nuevo hacia el otro lado. Mismo
  criterio que `established` para TCP, pero aplicado a ICMP.
- **IDS (Intrusion Detection System) / IPS (Intrusion Prevention System):** un equipo que
  **lee el contenido** del tráfico (no solo IP/puerto, como una ACL) y lo compara contra patrones
  de ataques conocidos. El IDS solo **avisa** (detecta); el IPS además **corta** el tráfico
  sospechoso en el momento (previene). Es la capa que le falta a una ACL: la ACL mira el "sobre"
  (origen, destino, puerto), el IDS/IPS mira el "contenido de la carta". Propuesto en
  `ANALISIS_SEGURIDAD.md` (consigna 5) — no se implementa en `RESOLUCION.pkt`, es un análisis
  escrito.
- **Syslog / servidor de logs centralizado:** protocolo (y servidor que lo recibe) para mandar
  **registros** (*logs*) de eventos de red — conexiones permitidas, bloqueos de ACL, cambios de
  config — desde varios equipos hacia un único lugar donde quedan guardados. Sin esto, cada equipo
  solo guarda su propio historial efímero (p.ej. los contadores de `show access-lists`, que ni
  sobreviven un reinicio si no se guardan). Propuesto en `ANALISIS_SEGURIDAD.md` (consigna 5) — no
  se implementa en `RESOLUCION.pkt`, es un análisis escrito.
- **Firewall con estado (*stateful*) vs. sin estado (*stateless*):** un firewall *stateful* (como
  la ASA) se acuerda de las conexiones que dejó pasar y permite automáticamente la respuesta (por
  ejemplo, si dejó salir un pedido web, deja entrar la respuesta sola, sin regla aparte). Una ACL
  común de router es *stateless*: no se acuerda de nada, cada paquete se evalúa solo, así que a
  veces hay que escribir reglas para el tráfico de ida *y* de vuelta. Es una de las diferencias
  entre usar un router con ACLs y un firewall dedicado — ver `FIREWALL_LICENSE_ISSUE.md`.

---

## 6. CLI de Cisco / ASA (lo mínimo para no perderte)

Los routers, switches y la ASA se configuran por texto (CLI). Hay **modos** anidados; el *prompt*
cambia según dónde estás:

| Prompt | Modo | Cómo se entra | Para qué |
|---|---|---|---|
| `Router>` | Usuario | (al abrir la CLI) | Solo mirar cosas básicas |
| `Router#` | Privilegiado (EXEC) | `enable` | Ver config, guardar, diagnosticar |
| `Router(config)#` | Configuración global | `configure terminal` | Cambiar la configuración |
| `Router(config-if)#` | Interfaz | `interface Gi0/1` | Configurar una boca |

Comandos que vas a repetir mucho:

- `enable` → entrar a modo privilegiado.
- `configure terminal` (o `conf t`) → entrar a configuración.
- `exit` → subir un nivel; `end` → volver directo a `#`.
- `no <comando>` → deshace/borra ese comando (p.ej. `no shutdown` prende una interfaz).
- `do show ...` → correr un `show` sin salir del modo config.
- **Guardar la config:** `write memory` (o `copy running-config startup-config`). Si no, se pierde
  al apagar.

Comandos de verificación (para revisiones):

- `show ip interface brief` → resumen de interfaces y sus IPs / estado (up/down).
- `show running-config` → toda la configuración actual.
- `show vlan brief` (switch) → qué VLANs existen y qué puertos tienen.
- `show interfaces trunk` (switch) → qué enlaces son trunk y qué VLANs pasan.
- En la ASA: `show nameif`, `show route`, `show access-list`, `show nat`.

---

## 7. Diagnóstico (probar que algo anda)

- **`ping <ip>`** (desde el Command Prompt de una PC o desde `#` en un equipo): ¿hay conectividad?
  Respuestas = OK; *Request timed out* / *Destination unreachable* = algo falla.
- **`tracert <ip>`** (PC) / **`traceroute`** (equipo): muestra por qué saltos pasa el tráfico; útil
  para ver dónde se corta.
- **Navegador (Desktop → Web Browser):** para probar HTTP/HTTPS contra el server (`http://IP`,
  `https://IP`).
- **Simulation mode** (esquina inferior derecha): reproduce el viaje de los paquetes paso a paso;
  sirve para *ver* dónde una ACL bloquea algo.

---

## 8. Cómo trabajamos las revisiones

Cuando termines una parte, mandame lo más útil para revisar:
- Una **captura** del lienzo o del cuadro que estés configurando, **o**
- El texto de `show running-config` / `show ip interface brief` del equipo, **o**
- Simplemente contame qué hiciste y qué resultado te dio un `ping`.

Con eso valido contra el objetivo de la fase (en `PLAN_FASES.md`) y te digo qué corregir o si
podemos avanzar.
