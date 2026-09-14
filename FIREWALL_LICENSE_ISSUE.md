# Por qué se reemplaza la ASA 5505 por el Router Cisco 4331

_(Términos técnicos: ver_ `GLOSARIO.md`_.)_

Una **interfaz** de red es una "puerta" del equipo, activada y con nombre, dedicada a una red
puntual. Esto vale sin importar cómo llegue el cableado: si 3 VLANs comparten un mismo cable
**trunk** (una técnica normal para llevar varias VLANs por un solo cable, usando una subinterfaz
por VLAN — es justo lo que hace el Router 4331 más abajo), cada una de esas 3 VLANs **igual
necesita su propia interfaz activada** del lado del equipo. El cable no es el límite; el límite es
cuántas interfaces puede tener activas el equipo en total.

Esta topología necesita **5 interfaces activas** en el equipo del medio (el que hace de firewall):

1. Una hacia Internet (`outside`)
2. Una hacia el servidor web (`dmz`)
3. Una para la VLAN 10 — Administración
4. Una para la VLAN 20 — Sistemas
5. Una para la VLAN 30 — Usuarios

Cisco vende sus equipos con distintos "planes" de funciones activadas, como un plan de celular:
según cuánto se paga, se habilitan más o menos funciones. La ASA 5505 que carga Packet Tracer
viene con el plan más básico, llamado **Base**, que impone una regla fija: **como máximo 3
interfaces activas a la vez en todo el equipo**, sin importar cuántos puertos físicos tenga ni
cómo estén cableados.

Ahí está el problema: se necesitan 5 interfaces y el plan Base solo permite 3. Las de `outside` y
`dmz` ya consumen 2 de esas 3, y queda **una sola libre** — pero hacen falta 3 más (una por VLAN).
No alcanza aunque las 3 VLANs lleguen juntas por un solo cable trunk: cada una necesita su propia
interfaz activada, y el cupo total del equipo es 3, no 5. No es un error de configuración: es un
límite de fábrica del equipo.

La consigna, en la lista de equipos, indica **"1 Router Cisco 4331 / Firewall ASA"**: permite usar
cualquiera de los dos para este rol. Por eso no se agrega un equipo de más — se elige la otra
opción que el enunciado ya contemplaba. El Router 4331 no tiene el límite de "3 puertas" porque no
es un producto con planes de licencia por función como la ASA.

> La ASA 5505 del simulador viene con licencia Base, que limita a 3 interfaces con nombre en
> total, y el diseño necesita 5. Como la consigna permite usar Router 4331 o ASA indistintamente,
> se migró el rol de firewall/gateway al Router 4331.

## Detalle técnico

En Fase 1, al configurar `dmz` en la ASA saltó:

```
ERROR: This license does not allow configuring more than 2 interfaces with nameif...
```

Pasaba porque `Vlan1` traía de fábrica `nameif inside` cargado, y sumado a `outside` ya eran 2
interfaces con nombre — `dmz` iba a ser la 3ª. Se resolvió liberando `Vlan1` (`no nameif` /
`no ip address`), dejando 2 cupos usados (`outside` + `dmz`).

La licencia **Base** de la ASA 5505 tiene un tope duro de **3 interfaces con nombre en total** — 2
libres + **1 restringida** (solo se habilita agregando `no forward interface <otra>`, que le
impide iniciar tráfico hacia una de las otras dos). Con `outside`+`dmz` ocupando 2, queda 1 cupo
restringido — no alcanza para 3 gateways de VLAN.

Se migra el rol de perímetro + inter-VLAN al **Router Cisco 4331**. Cada VLAN se resuelve con una
**subinterfaz** (`router-on-a-stick`) sobre el mismo puerto físico trunk, sin tope de licencia.

## Tradeoff: router con ACLs vs. firewall dedicado

Un router no es exactamente lo mismo que un firewall dedicado como la ASA. Diferencias reales:

- **Sin bloqueo automático (_default-deny_):** la ASA bloquea por defecto tráfico de una red menos
  confiable hacia una más confiable. Un router IOS deja pasar todo entre sus interfaces salvo que
  una ACL lo prohíba explícitamente — si falta una regla, ese tráfico pasa. Toda la
  responsabilidad de "denegar lo no permitido" recae en las ACLs que se escriban en Fase 4.
- **Sin memoria de conexiones (_stateless_):** la ASA es _stateful_ — se acuerda de las conexiones
  que dejó pasar y permite la respuesta sola. Las ACLs de IOS son _stateless_: no se acuerdan de
  nada, hay que permitir explícitamente el tráfico de ida y de vuelta.
- **Sin inspección de protocolo:** la ASA entiende protocolos como FTP (por ejemplo, abre sola el
  canal de datos dinámico que arma una sesión FTP activa). El router con ACLs solo filtra por
  IP/puerto/protocolo, no entiende el contenido del protocolo.
- **NAT aparte:** en la ASA, NAT está integrado con la política de seguridad. En IOS es un
  subsistema aparte (`ip nat inside`/`outside`) que hay que coordinar a mano con las ACLs.
- **Gestión y alta disponibilidad:** la ASA tiene ASDM (interfaz gráfica) y failover (con Security
  Plus). El 4331 se gestiona todo por CLI.

El 4331 con ACLs es un router con función de filtrado perimetral (se lo suele llamar _"screening
router"_) — válido y común en el mundo real, pero no reemplaza en robustez a un firewall dedicado.
Es un argumento aprovechable para la consigna punto 5 (justificar agregar un firewall/UTM o IPS
dedicado en una implementación real).
