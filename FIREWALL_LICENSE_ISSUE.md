# Por qué se reemplaza la ASA 5505 por el Router Cisco 4331

## El problema

En Fase 1, al configurar `dmz` en la ASA saltó:
```
ERROR: This license does not allow configuring more than 2 interfaces with nameif...
```
Causa: `Vlan1` traía de fábrica `nameif inside`, sumado a `outside` ya eran 2 — `dmz` sería la 3ª.
Se resolvió liberando `Vlan1` (`no nameif` / `no ip address`), dejando 2 cupos usados.

**Por qué no alcanza para Fase 2:** la licencia **Base** de la ASA 5505 tiene un tope duro de **3
interfaces con nombre en total** (2 libres + 1 restringida, que solo se habilita con
`no forward interface <otra>`). `outside` + `dmz` ya ocupan 2 de esos 3. Queda **1 cupo**
disponible — pero Fase 2 necesita gateway propio para **3 VLANs** (10, 20, 30). Liberar `Vlan1`
recuperó 1 cupo; acá hacen falta 3.

## La decisión

Reemplazar la ASA 5505 por el **Router Cisco 4331** en el rol de perímetro + inter-VLAN. La
consigna los da como equivalentes ("1 Router Cisco 4331 / Firewall ASA"), así que no se agrega
ningún dispositivo fuera de lo pedido. El 4331 resuelve cada VLAN con una **subinterfaz** IOS
(`Gi0/0/2.10`, `.20`, `.30`) sobre el mismo puerto trunk, sin tope de licencia.

## El tradeoff: router con ACLs vs. firewall dedicado

- **Default-deny vs. default-permit:** la ASA bloquea por defecto tráfico de menor a mayor
  security-level. Un router IOS reenvía entre todas sus interfaces por defecto — si falta una ACL
  en algún sentido, ese tráfico pasa. Toda la responsabilidad de "denegar lo no permitido" recae en
  las ACLs que se escriban en Fase 4.
- **Stateful vs. stateless:** la ASA trackea conexiones y deja pasar el retorno automáticamente.
  Las ACLs extendidas de IOS no entienden estado (`established` en TCP es una aproximación, no
  inspección real). IOS tiene CBAC/Zone-Based Policy Firewall para esto, pero es config aparte, no
  algo activado por diseño.
- **Inspección de aplicación:** la ASA entiende protocolos (FTP, HTTP, ICMP) y abre puertos
  dinámicos de sesión. El router con ACLs solo filtra IP/puerto/protocolo.
- **NAT:** en la ASA está integrado a la política de seguridad; en IOS es un subsistema aparte
  (`ip nat inside`/`outside`) que hay que coordinar a mano con las ACLs.
- **Gestión y HA:** ASDM y failover (Security Plus) en la ASA; el 4331 es todo CLI, sin ese nivel
  de alta disponibilidad.

**Síntesis:** el 4331 con ACLs es un router con función de filtrado perimetral ("screening
router") — válido y común, pero no reemplaza en robustez a un firewall dedicado. Argumento
aprovechable para la consigna punto 5 (justificar agregar un firewall/UTM o IPS dedicado).
