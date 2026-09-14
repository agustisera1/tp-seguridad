# Plan de fases — TP1 Seguridad (`RESOLUCION.pkt`)

## Contexto

El TP1 pide construir en Packet Tracer una red con **firewall ASA 5505**, salida a Internet
vía **Router ISP**, tres VLANs internas y una **DMZ** con servidor web, aplicando **ACLs con
mínimo privilegio**. La consigna y la topología objetivo se extrajeron de `CONSIGNA.pdf`.

El archivo `clase_2.pkt` de la clase usaba un diseño distinto (router-on-a-stick con dos 1941 +
OSPF, direccionamiento 172.16/200.1.1.0) que **no encaja** con el diseño ASA/DMZ del TP.
**Decisión: se descartó `clase_2.pkt` y se construye `RESOLUCION.pkt` desde cero.**

## Perfil del usuario y modo de ejecución

Ejecución **guiada y asistida** (el usuario está aprendiendo Packet Tracer y los CLI de Cisco/ASA):

- En cada fase se entregan: (a) **pasos en la GUI de Packet Tracer** (qué dispositivo, qué cable,
  qué puerto) y (b) **configuración CLI lista para copiar/pegar**, comentada.
- El usuario cablea/configura en el programa y **pide revisiones**; se valida el objetivo de la
  fase antes de avanzar.
- Material de apoyo: **`GLOSARIO.md`** (componentes + conceptos + CLI básico).

## Alineación con la consigna (evitar "entregar de más")

Las 5 fases son *proceso* (reparto gradual del mismo trabajo), no entregables extra. Mapean 1:1
con la consigna. Se marca lo que la cátedra **no pide**:

| Ítem | ¿Lo pide? | En el plan |
|---|---|---|
| Topología según imagen (1) | Sí | Fase 1 — core |
| IP/máscara/GW a todo (2) | Sí | Fase 1 — core |
| Comunicación entre VLANs (3) | Sí | Fase 2 — core |
| ACLs por segmento + mínimo privilegio (4) | Sí | Fase 4 — core |
| Acceso de "Externos" al server (4) | Sí | Fase 3 — core (NAT) |
| Análisis de seguridad adicional (5) | Sí | Fase 5 — core (escrito) |
| Informe PDF + .pkt + .zip (entrega) | Sí | Fase 5 — core |
| Hardening (SSH admin, claves cifradas, banner) | **No** | Fase 5 — **OPCIONAL** |
| Navegación interna → Internet | **No** (solo hace falta para Externos) | Fase 3 — **OPCIONAL** |

## Decisiones tomadas

1. **Archivo:** nuevo `RESOLUCION.pkt`, desde cero. Se desestima `clase_2.pkt`.
2. **Perímetro + inter-VLAN — MIGRACIÓN CONFIRMADA:** la ASA 5505 se **reemplaza por el Router
   Cisco 4331** para este rol (la consigna los da como equivalentes: "1 Router Cisco 4331 /
   Firewall ASA"). Motivo, matemática de la limitante de licencia y el tradeoff router-vs-firewall:
   ver **`FIREWALL_LICENSE_ISSUE.md`**. Pasos GUI+CLI de la migración: `EJECUCION.md` (Fase 2).

## Topología e inventario objetivo (según la imagen de la consigna)

```
        INTERNET (nube)
            |
        Router ISP  200.10.10.1/30
            | (outside 200.10.10.2/30)
      Router 4331 ───────── dmz 192.168.40.1/24 ── SW-DMZ ── WEB-SERVER 192.168.40.10/24
            | (subinterfaces 802.1Q, trunk hacia SW-LAN)
          SW-LAN
        /   |   \
   VLAN10 VLAN20 VLAN30
   ADMIN  SISTEMAS USUARIOS
  PC-ADM1 PC-SIS1 PC-USR1
```

| Dispositivo | Rol | Notas |
|---|---|---|
| Router ISP | Borde hacia Internet | 200.10.10.1/30; enlace a la nube Internet |
| Router 4331 | Perímetro + inter-VLAN + firewall (ACLs) | outside .2/30, subif VLAN10/20/30 (trunk a SW-LAN), dmz .40.1/24 |
| SW-LAN (2960) | Acceso interno | VLAN 10/20/30, puertos de acceso + trunk al Router 4331 |
| SW-DMZ (2960) | Acceso DMZ | Conecta WEB-SERVER |
| WEB-SERVER | Servidor de servicios | HTTP/HTTPS/FTP (+SSH ver riesgo) |
| PC-ADM1 / PC-SIS1 / PC-USR1 | Clientes por segmento | Uno por VLAN |
| PC-EXT (Externos) | Cliente en Internet | Para probar acceso externo a la DMZ |
| Nube Internet | Simula Internet | — |

## Plan de direccionamiento

| Segmento | VLAN | Red / Máscara | Gateway | Host de prueba |
|---|---|---|---|---|
| Administración | 10 | 192.168.10.0 /24 | 192.168.10.1 (subif Router 4331) | PC-ADM1 .10.10 |
| Sistemas | 20 | 192.168.20.0 /24 | 192.168.20.1 (subif Router 4331) | PC-SIS1 .20.10 |
| Usuarios | 30 | 192.168.30.0 /24 | 192.168.30.1 (subif Router 4331) | PC-USR1 .30.10 |
| DMZ | — | 192.168.40.0 /24 | 192.168.40.1 (Router 4331, dmz) | WEB-SERVER .40.10 |
| WAN | — | 200.10.10.0 /30 | ISP .1 / Router 4331 outside .2 | — |
| Internet (prueba) | — | 100.100.100.0 /30 | Router ISP .1 | PC-EXT .2 |

*(La red `100.100.100.0/30` no la da la consigna — es una subred elegida en Fase 3 solo para poder
conectar a PC-EXT más allá del Router ISP y así probar el acceso de "Externos"; no forma parte del
direccionamiento oficial del TP.)*

El Router 4331 (IOS) no tiene security-levels: no hay default-deny implícito entre interfaces
como en la ASA. Todo el control de acceso entre segmentos se implementa a pulso con **ACLs
extendidas por interfaz** en Fase 4 — ver tradeoff en `FIREWALL_LICENSE_ISSUE.md`.

## Matriz de ACLs (consigna 4 — mínimo privilegio, destino = WEB-SERVER salvo aclaración)

| Origen | Permitido | Denegado (resto) |
|---|---|---|
| Administración | HTTPS, FTP | todo lo demás |
| Usuarios | HTTPS | todo lo demás |
| Sistemas | HTTPS, HTTP, FTP, SSH, ICMP **+ redes Usuarios y Admin** | todo lo demás |
| Externos (Internet) | HTTPS, FTP (subir archivos) → DMZ publicada por NAT | todo lo demás |

*(Implementación en Fase 4: ACLs extendidas nombradas de IOS en las subinterfaces del Router
4331, no `access-list`/`object-group` de sintaxis ASA.)*

## Fases (graduales, cada una deja la red en estado verificable)

Cada fase: se dan pasos GUI + CLI listo para pegar → el usuario lo hace en PT → pide revisión →
se valida el "estado al terminar" antes de pasar a la siguiente.

### Fase 1 — Topología física + direccionamiento base  *(consignas 1 y 2)*
- Crear `RESOLUCION.pkt`; colocar y cablear todos los dispositivos según la imagen.
- Asignar IP/máscara/GW a PCs, WEB-SERVER, Router ISP e interfaces de la ASA.
- SW-LAN: crear VLAN 10/20/30, puertos de acceso a cada PC, **trunk hacia la ASA**.
- SW-DMZ: puerto de acceso al WEB-SERVER.
- **Estado al terminar:** ping dentro de cada VLAN y de cada host a su gateway. Enlaces up.

### Fase 2 — Migración a Router 4331 + Inter-VLAN  *(consigna 3)*
- Reemplazar la ASA 5505 por el **Router Cisco 4331** (motivo: límite de licencia, ver
  Decisiones). Recablear los 3 enlaces existentes (Router ISP, SW-DMZ, SW-LAN) al nuevo
  dispositivo.
- Router 4331: reconfigurar `outside` y `dmz` (antes en la ASA) + subinterfaces `.10/.20/.30`
  (dot1Q) sobre el puerto trunk hacia SW-LAN, como gateway de cada VLAN.
- **Estado al terminar:** todas las VLANs se pingean entre sí, alcanzan la DMZ, y el enlace WAN
  hacia Router ISP sigue up. Todavía sin ACLs restrictivas (eso es Fase 4).

### Fase 3 — Salida a Internet y publicación de DMZ  *(ruteo + NAT)*
- PC-EXT conectado vía la Nube Internet a un 2do puerto de Router ISP (red de prueba
  `100.100.100.0/30`, ver nota en el Plan de direccionamiento).
- Router 4331: ruta por defecto hacia Router ISP.
- **NAT estático** (con `interface Gi0/0/2` como IP pública) para publicar el WEB-SERVER —
  puertos 443 (HTTPS) y 20/21 (FTP).
- ACL de entrada en `outside` permitiendo Internet→DMZ solo HTTPS/FTP (resto denegado).
- *(Opcional, no incluido)* PAT para navegación interna→Internet.
- **Estado al terminar:** PC-EXT llega al server publicado por HTTPS y FTP; HTTP y el resto
  quedan bloqueados. Instructivo completo en `EJECUCION.md`.

### Fase 4 — Políticas de seguridad / ACLs con mínimo privilegio  *(consigna 4)*
- Implementar la matriz de ACLs completa por segmento.
- Ajustar servicios del WEB-SERVER (HTTP/HTTPS/FTP) para demostrar permitido vs. bloqueado.
- **Estado al terminar:** la matriz permitido/bloqueado se comporta según la tabla (p.ej. HTTPS
  OK y HTTP bloqueado para Usuarios).

### Fase 5 — Análisis, verificación final y entrega  *(consigna 5 + entrega)*
- **Consigna 5 (análisis escrito):** justificar dispositivos de seguridad adicionales (IDS/IPS,
  AAA, WAF…), ubicación y función.
- Batería de pruebas con capturas (accesos permitidos/bloqueados) para el informe.
- Producir **informe PDF** + `RESOLUCION.pkt` final; comprimir en `.zip` (`TP1_SSI_ApellidoNombre.zip`).
- *(Opcional)* Hardening: `enable secret`, `service password-encryption`, SSH, banner MOTD.

## Riesgos / puntos a resolver durante la ejecución
- **ASA 5505 licencia Base (RESUELTO):** migración a Router 4331. Detalle y tradeoff en
  `FIREWALL_LICENSE_ISSUE.md`.
- **SSH al WEB-SERVER:** el Server-PT no ofrece SSH nativo; se decide en Fase 4 si se permite
  igual a nivel ACL o se representa contra un dispositivo de red en la DMZ.
- **"Externos":** requiere NAT estático (sintaxis `ip nat` de IOS en vez de la de ASA) + ACL en
  outside; validado con PC-EXT en Fase 3.

## Verificación end-to-end
Al cerrar cada fase se corre su chequeo (ping/tracert/servicios). Verificación global final:
matriz de accesos por segmento (permitidos y bloqueados) con capturas, más conectividad interna,
inter-VLAN, DMZ y acceso de Externos funcionando.

## Estado y próximos pasos
- [x] Plan y documentación (este archivo + `GLOSARIO.md`).
- [x] **Fase 1** — completa (topología, IPs, VLANs/trunk en SW-LAN). **Errata:** la Tarea 4 (ASA
      `outside`/`dmz`) queda superada por la migración a Router 4331 — se rehace al arrancar
      Fase 2, ver `EJECUCION.md`.
- [x] **Fase 2** — completa (migración ASA→4331, `outside`/`dmz`/subinterfaces VLAN10/20/30,
      trunk SW-LAN confirmado, ping de gateway e inter-VLAN OK). Detalle en `EJECUCION.md`.
- [ ] **Fase 3** — salida a Internet y publicación de DMZ (NAT). Instructivo GUI+CLI completo en
      `EJECUCION.md`, listo para ejecutar.
- [ ] Fases 4 y 5.
