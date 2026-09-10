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
| **Router ISP** | Router que simula al proveedor de Internet | Borde WAN; enlaza la nube Internet con la ASA (red 200.10.10.0/30) |
| **ASA 5505** | *Firewall* de Cisco | El corazón de seguridad: separa outside (Internet) / inside (LAN) / dmz, y aplica las ACLs |
| **Switch 2960** | Switch de capa 2 (24 bocas) | SW-LAN reparte las VLANs internas; SW-DMZ conecta el servidor |
| **Server-PT** | Servidor genérico | El WEB-SERVER: ofrece HTTP/HTTPS/FTP a probar |
| **PC** | Computadora cliente | Un host por segmento (Admin, Sistemas, Usuarios) + uno "Externo" en Internet |
| **Nube (Cloud)** | Representa Internet | Extremo remoto; de ahí "vienen" los Externos |

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

- **FastEthernet (Fa0/1):** boca de 100 Mbps. Los switches 2960 tienen Fa0/1…Fa0/24.
- **GigabitEthernet (Gi0/1):** boca de 1 Gbps. Los 2960 tienen Gi0/1 y Gi0/2 (suelen usarse para
  el *uplink* / trunk hacia el router o la ASA).
- **Ethernet0/x (ASA):** las bocas de la ASA 5505 se llaman `Ethernet0/0`, `0/1`, … Por convención
  `Ethernet0/0` = outside y `Ethernet0/1` = inside.
- **Subinterfaz (Gi0/1.10):** una interfaz "virtual" sobre una física, atada a una VLAN. La usa
  la ASA para ser gateway de cada VLAN por el mismo cable (trunk).

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
  (en el TP, la ASA con sus subinterfaces).
- **ACL (Access Control List):** lista de reglas *permitir/denegar* tráfico según origen, destino
  y servicio. Es el núcleo de la consigna 4.
- **Mínimo privilegio:** dar solo los permisos estrictamente necesarios; todo lo no permitido, se
  niega. Las ACL terminan con un "deny" implícito que ayuda a esto.
- **DMZ (zona desmilitarizada):** red intermedia donde se ponen los servidores accesibles desde
  afuera (el WEB-SERVER), separada de la LAN interna para que un ataque al server no toque la LAN.
- **NAT / PAT:** traducción de direcciones. **NAT estático** publica el server privado
  (192.168.40.10) con una IP pública para que los Externos lo alcancen. **PAT** es el NAT "de
  muchos a uno" que usan las PCs para salir a Internet.
- **security-level (ASA):** número 0–100 por interfaz. Más alto = más confiable. Por defecto el
  tráfico va de mayor a menor nivel; de menor a mayor (Internet→DMZ) hay que permitirlo explícito.
  En el TP: outside 0, dmz 50, inside 100.

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
