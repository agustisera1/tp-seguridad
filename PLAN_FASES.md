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
2. **Inter-VLAN + firewall — PIVOT CONFIRMADO (Fase 2):** la ASA 5505 de este `.pkt` corre con
   **licencia Base** (tope ~2 `nameif` completas + 1 restringida; sin soporte de trunk/subinterfaces
   802.1Q, eso es Security Plus). Con `outside`+`dmz` ya ocupando las 2 completas, no hay margen
   para 3 subinterfaces VLAN10/20/30 más. Se descarta la idea original ("ASA hace todo") y se
   pivotea: un **switch L3 (3560, "SW-L3")** hace el inter-VLAN (SVIs como gateway de cada VLAN),
   y la ASA queda solo para **perímetro + DMZ**, con un 3er `nameif` `inside` como enlace routeado
   punto a punto hacia SW-L3 (no trunk). Detalle y justificación en `EJECUCION.md` Fase 2.

## Topología e inventario objetivo (según la imagen de la consigna)

```
        INTERNET (nube)
            |
        Router ISP  200.10.10.1/30
            | (outside 200.10.10.2/30)
          ASA 5505 ───────── dmz 192.168.40.1/24 ── SW-DMZ ── WEB-SERVER 192.168.40.10/24
            | (inside 192.168.50.1/30, enlace routeado punto a punto, SIN trunk)
          SW-L3 (192.168.50.2/30) ── SVIs Vlan10/20/30 = gateway de cada VLAN
            | (trunk 802.1Q)
          SW-LAN
        /   |   \
   VLAN10 VLAN20 VLAN30
   ADMIN  SISTEMAS USUARIOS
  PC-ADM1 PC-SIS1 PC-USR1
```

| Dispositivo | Rol | Notas |
|---|---|---|
| Router ISP | Borde hacia Internet | 200.10.10.1/30; enlace a la nube Internet |
| ASA 5505 | Firewall perímetro + DMZ | outside .2/30, inside .50.1/30 (punto a punto a SW-L3, `no forward interface outside`), dmz .40.1/24 |
| SW-L3 (3560) | Inter-VLAN (routing) | `ip routing`, SVIs Vlan10/20/30 = gateway de cada VLAN, puerto routeado .50.2/30 hacia ASA, trunk hacia SW-LAN |
| SW-LAN (2960) | Acceso interno | VLAN 10/20/30, puertos de acceso + trunk hacia SW-L3 |
| SW-DMZ (2960) | Acceso DMZ | Conecta WEB-SERVER |
| WEB-SERVER | Servidor de servicios | HTTP/HTTPS/FTP (+SSH ver riesgo) |
| PC-ADM1 / PC-SIS1 / PC-USR1 | Clientes por segmento | Uno por VLAN |
| PC-EXT (Externos) | Cliente en Internet | Para probar acceso externo a la DMZ |
| Nube Internet | Simula Internet | — |

## Plan de direccionamiento

| Segmento | VLAN | Red / Máscara | Gateway | Host de prueba |
|---|---|---|---|---|
| Administración | 10 | 192.168.10.0 /24 | 192.168.10.1 (SVI en SW-L3) | PC-ADM1 .10.10 |
| Sistemas | 20 | 192.168.20.0 /24 | 192.168.20.1 (SVI en SW-L3) | PC-SIS1 .20.10 |
| Usuarios | 30 | 192.168.30.0 /24 | 192.168.30.1 (SVI en SW-L3) | PC-USR1 .30.10 |
| DMZ | — | 192.168.40.0 /24 | 192.168.40.1 (ASA dmz) | WEB-SERVER .40.10 |
| WAN | — | 200.10.10.0 /30 | ISP .1 / ASA outside .2 | — |
| Transit ASA↔SW-L3 | — | 192.168.50.0 /30 | ASA inside .50.1 / SW-L3 .50.2 | — |

Security-levels ASA: `outside` 0, `dmz` 50, `inside`/VLANs 100.

## Matriz de ACLs (consigna 4 — mínimo privilegio, destino = WEB-SERVER salvo aclaración)

| Origen | Permitido | Denegado (resto) |
|---|---|---|
| Administración | HTTPS, FTP | todo lo demás |
| Usuarios | HTTPS | todo lo demás |
| Sistemas | HTTPS, HTTP, FTP, SSH, ICMP **+ redes Usuarios y Admin** | todo lo demás |
| Externos (Internet) | HTTPS, FTP (subir archivos) → DMZ publicada por NAT | todo lo demás |

## Fases (graduales, cada una deja la red en estado verificable)

Cada fase: se dan pasos GUI + CLI listo para pegar → el usuario lo hace en PT → pide revisión →
se valida el "estado al terminar" antes de pasar a la siguiente.

### Fase 1 — Topología física + direccionamiento base  *(consignas 1 y 2)*
- Crear `RESOLUCION.pkt`; colocar y cablear todos los dispositivos según la imagen.
- Asignar IP/máscara/GW a PCs, WEB-SERVER, Router ISP e interfaces de la ASA.
- SW-LAN: crear VLAN 10/20/30, puertos de acceso a cada PC, **trunk hacia la ASA**.
- SW-DMZ: puerto de acceso al WEB-SERVER.
- **Estado al terminar:** ping dentro de cada VLAN y de cada host a su gateway. Enlaces up.

### Fase 2 — Inter-VLAN y conectividad L3 interna  *(consigna 3)*
- Agregar **SW-L3** (switch 3560), recablear el trunk de SW-LAN hacia SW-L3 (en vez de a la ASA),
  y crear un enlace routeado punto a punto SW-L3↔ASA (`inside`, transit 192.168.50.0/30).
- SW-L3: `ip routing` + SVIs Vlan10/20/30 (gateway de cada VLAN) + ruta por defecto hacia la ASA.
- ASA: 3er `nameif` `inside` (Vlan1, sin trunk) + `no forward interface outside` (Base license
  permite 3 VLANs pero la 3ra no puede iniciar tráfico hacia dos interfaces a la vez; se restringe
  el salto directo inside→outside, que de todos modos es opcional) + rutas estáticas hacia las
  redes VLAN vía SW-L3.
- **Estado al terminar:** todas las VLANs se pingean entre sí, alcanzan la DMZ y la interfaz
  `inside` de la ASA.

### Fase 3 — Salida a Internet y publicación de DMZ  *(ruteo + NAT)*
- Router ISP configurado; ruta por defecto ASA→ISP.
- **NAT estático** para publicar el WEB-SERVER hacia `outside` (acceso de "Externos").
- ACL de entrada en `outside` permitiendo Internet→DMZ solo HTTPS/FTP.
- *(Opcional)* PAT para navegación interna→Internet.
- **Estado al terminar:** PC-EXT llega al server publicado (HTTPS/FTP).

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
- **ASA 5505 licencia Base (RESUELTO en Fase 1/2):** no soporta trunk/subinterfaces ni más de ~2-3
  `nameif`. Se pivotea el inter-VLAN a SW-L3 (switch 3560), dejando la ASA solo para perímetro+DMZ.
  Riesgo abierto: el 3er `nameif` (`inside`) podría chocar igual con la restricción de licencia al
  intentar hablar con `dmz` y `outside` simultáneamente — se prueba `no forward interface outside`
  como mitigación; si sigue fallando, documentar en `EJECUCION.md` y decidir alternativa ahí.
- **SSH al WEB-SERVER:** el Server-PT no ofrece SSH nativo; se decide en Fase 4 si se permite
  igual a nivel ACL o se representa contra un dispositivo de red en la DMZ.
- **"Externos":** requiere NAT estático + ACL en outside; validado con PC-EXT en Fase 3.

## Verificación end-to-end
Al cerrar cada fase se corre su chequeo (ping/tracert/servicios). Verificación global final:
matriz de accesos por segmento (permitidos y bloqueados) con capturas, más conectividad interna,
inter-VLAN, DMZ y acceso de Externos funcionando.

## Estado y próximos pasos
- [x] Plan y documentación (este archivo + `GLOSARIO.md`).
- [x] **Fase 1** — completa. Detalle paso a paso en `EJECUCION.md`.
- [ ] **Fase 2** — pivot a switch L3 confirmado (ver arriba); pasos detallados en `EJECUCION.md`.
- [ ] Fases 3 a 5.
