# Análisis de seguridad adicional — Consigna 5

> Consigna: *"Analice la topología de red. Si usted fuese el responsable de seguridad de la
> empresa, ¿qué dispositivo o dispositivos de seguridad adicionales incorporaría a la
> infraestructura? Indique dónde los ubicaría, qué función cumplirían y justifique técnicamente su
> elección."*

Este es un análisis **escrito**, no se implementa nada de esto en `RESOLUCION.pkt` — la red que
armamos en las Fases 1 a 4 queda tal cual. Acá se responde "qué agregaría si esto fuera una red
real", con foco en las limitaciones concretas que quedan expuestas por el diseño ya construido.

## Punto de partida: qué protege (y qué no) lo que ya tenemos

El router del firewall filtra tráfico con **ACLs** (Fase 4): decide sí/no según de dónde viene el
tráfico, hacia dónde va y por qué puerto. Es un control efectivo, pero tiene un límite por diseño:
una ACL nunca abre el paquete para ver **qué hay adentro** — solo mira la "etiqueta" (IP origen, IP
destino, puerto). Si un ataque viaja disfrazado de tráfico permitido (por ejemplo, un intento de
explotar una vulnerabilidad del servidor web, enviado por HTTPS al puerto 443, que es un puerto que
nuestra propia ACL deja pasar a propósito), el router lo deja cruzar igual: para la ACL es tráfico
válido.

Tampoco tenemos, hoy, ningún lugar donde quede un **registro histórico** de lo que pasó: si algo se
bloqueó o se intentó, la única evidencia es lo que se ve en el momento con `show access-lists`
(contadores que ni siquiera sobreviven un reinicio si no se guardaron).

Esos son los dos agujeros que se cubren con las dos propuestas siguientes.

---

## 1. IDS/IPS — inspección de contenido en la DMZ

**Qué es:** un IDS (*Intrusion Detection System*) es un equipo que **lee el contenido** del tráfico
que pasa (no solo IP/puerto, como una ACL) y lo compara contra patrones de ataques conocidos; si
encuentra algo sospechoso, avisa. Un IPS (*Intrusion Prevention System*) hace lo mismo pero además
puede **cortar el tráfico** en el momento, sin esperar a que alguien reaccione a la alerta. La
diferencia con una ACL es de profundidad: la ACL mira el "sobre" (origen/destino/puerto), el
IDS/IPS mira el "contenido de la carta".

**Dónde lo ubicaría:** en línea, entre el router del firewall (interfaz `dmz`,
`GigabitEthernet0/0/1`) y el SW-DMZ — es decir, justo antes de que el tráfico llegue al
WEB-SERVER. Es el punto exacto por donde pasa **todo** el tráfico que entra desde Internet
(Externos, vía NAT) y también el de Administración/Usuarios/Sistemas hacia el server: el segmento
más expuesto de toda la red, porque es el único alcanzable desde afuera de la organización.

**Función que cumple:** analiza el tráfico HTTPS/HTTP/FTP que ya dejaron pasar las ACLs y busca
patrones de ataque a nivel aplicación (por ejemplo, intentos de explotación contra el servidor
web) que una ACL no puede ver porque no mira el contenido del paquete, solo su "etiqueta".

**Justificación técnica:** las ACLs de Fase 4 y `ACL-OUTSIDE-IN` de Fase 3 funcionan a nivel
IP/puerto (capas 3 y 4) — son necesarias pero no suficientes, porque cualquier tráfico que use un
puerto permitido (443, 80, 21) entra igual, sea legítimo o malicioso. El IDS/IPS agrega la capa que
falta: inspección de contenido, específicamente en el único segmento con exposición directa a
Internet.

---

## 2. Servidor de logs centralizado (Syslog)

**Qué es:** un servidor que recibe y guarda, en un solo lugar, los **registros** (*logs*) que
generan otros equipos de la red — el router del firewall, los switches — cada vez que pasa algo
relevante (una conexión permitida, una bloqueada por ACL, un cambio de configuración). El protocolo
que se usa para mandar esos registros se llama **Syslog**.

**Dónde lo ubicaría:** en la VLAN 20 (Sistemas), colgado de SW-LAN como cualquier otro host de esa
VLAN. Sistemas ya es, en la matriz de ACLs de Fase 4, el segmento con más privilegios de la red
interna (acceso ampliado al WEB-SERVER + a las redes de Administración y Usuarios) — es coherente
que ahí viva la infraestructura de monitoreo, junto al resto de las herramientas de administración
de la red.

**Función que cumple:** el router del firewall (y los switches) se configurarían para mandarle una
copia de sus eventos a este servidor (`logging <ip-del-servidor>` en IOS). Así queda un historial
centralizado y persistente de la actividad de la red, sin depender de la memoria volátil de cada
equipo.

**Justificación técnica:** hoy, la única evidencia de que una ACL bloqueó algo es un contador
efímero (`show access-lists`) en el propio router — no hay forma de reconstruir, días después,
quién intentó qué y cuándo. Para responder a un incidente real hace falta un registro que
sobreviva al reinicio de un equipo y que junte los eventos de **toda** la red en un solo lugar, no
equipo por equipo. Es, además, el complemento natural del IDS/IPS: ese equipo genera alertas, y
sin un lugar central donde guardarlas, se pierden.

---

## Resumen

| Dispositivo | Ubicación | Cubre |
|---|---|---|
| IDS/IPS | En línea, entre el router del firewall (`dmz`) y SW-DMZ | Las ACLs no inspeccionan contenido — un ataque disfrazado de tráfico permitido pasa igual |
| Servidor de Syslog | VLAN 20 (Sistemas) | No existe hoy un registro histórico/centralizado de lo que la red permitió o bloqueó |
