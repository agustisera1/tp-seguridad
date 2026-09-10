# Plan de fases — TP1 Seguridad (`RESOLUCION.pkt`)

## Contexto

El TP1 pide construir en Packet Tracer una red con **firewall ASA 5505**, salida a Internet
vía **Router ISP**, tres VLANs internas y una **DMZ** con servidor web, aplicando **ACLs con
mínimo privilegio**. La consigna y la topología objetivo se extrajeron de `CONSIGNA.pdf`.

El archivo `clase_2.pkt` (ver `ESTADO_ACTUAL.md`) usa un diseño distinto (router-on-a-stick con
dos 1941 + OSPF, direccionamiento 172.16/200.1.1.0) que **no encaja** con el diseño ASA/DMZ del
TP. **Decisión: se descarta `clase_2.pkt` y se construye `RESOLUCION.pkt` desde cero.**

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
2. **Inter-VLAN + firewall:** la **ASA 5505 hace todo**, fiel a la imagen — es gateway de las 3
   VLANs mediante **subinterfaces 802.1Q sobre `inside`** (trunk al SW-LAN) y maneja `outside`,
   `dmz` y las ACLs.
   - **Checkpoint de contingencia (Fase 2):** si la ASA 5505 de Packet Tracer no soporta
     trunk/subinterfaces, se pivotea a inter-VLAN con **switch L3 / router interno**, dejando la
     ASA solo para perímetro+DMZ. No cambia las fases 3–5.

## Topología e inventario objetivo (según la imagen de la consigna)

```
        INTERNET (nube)
            |
        Router ISP  200.10.10.1/30
            | (outside 200.10.10.2/30)
          ASA 5505 ───────── dmz 192.168.40.1/24 ── SW-DMZ ── WEB-SERVER 192.168.40.10/24
            | (inside, trunk 802.1Q)
          SW-LAN
        /   |   \
   VLAN10 VLAN20 VLAN30
   ADMIN  SISTEMAS USUARIOS
  PC-ADM1 PC-SIS1 PC-USR1
```

| Dispositivo | Rol | Notas |
|---|---|---|
| Router ISP | Borde hacia Internet | 200.10.10.1/30; enlace a la nube Internet |
| ASA 5505 | Firewall perímetro + inter-VLAN | outside .2/30, inside (subif VLAN10/20/30), dmz .40.1/24 |
| SW-LAN (2960) | Acceso interno | VLAN 10/20/30, puertos de acceso + trunk a ASA |
| SW-DMZ (2960) | Acceso DMZ | Conecta WEB-SERVER |
| WEB-SERVER | Servidor de servicios | HTTP/HTTPS/FTP (+SSH ver riesgo) |
| PC-ADM1 / PC-SIS1 / PC-USR1 | Clientes por segmento | Uno por VLAN |
| PC-EXT (Externos) | Cliente en Internet | Para probar acceso externo a la DMZ |
| Nube Internet | Simula Internet | — |

## Plan de direccionamiento

| Segmento | VLAN | Red / Máscara | Gateway | Host de prueba |
|---|---|---|---|---|
| Administración | 10 | 192.168.10.0 /24 | 192.168.10.1 (ASA sub-if) | PC-ADM1 .10.10 |
| Sistemas | 20 | 192.168.20.0 /24 | 192.168.20.1 (ASA sub-if) | PC-SIS1 .20.10 |
| Usuarios | 30 | 192.168.30.0 /24 | 192.168.30.1 (ASA sub-if) | PC-USR1 .30.10 |
| DMZ | — | 192.168.40.0 /24 | 192.168.40.1 (ASA dmz) | WEB-SERVER .40.10 |
| WAN | — | 200.10.10.0 /30 | ISP .1 / ASA outside .2 | — |

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
- ASA: subinterfaces `inside.10/.20/.30` (dot1Q) como gateways; permitir tránsito entre VLANs
  antes de restringir con ACLs.
- **Checkpoint de contingencia:** verificar que la ASA 5505 de PT soporta trunk/subif; si no,
  pivotear a switch L3 / router interno.
- **Estado al terminar:** todas las VLANs se pingean entre sí y alcanzan la DMZ y la ASA.

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
- **ASA 5505 trunk/subif en PT:** contingencia prevista en Fase 2.
- **SSH al WEB-SERVER:** el Server-PT no ofrece SSH nativo; se decide en Fase 4 si se permite
  igual a nivel ACL o se representa contra un dispositivo de red en la DMZ.
- **"Externos":** requiere NAT estático + ACL en outside; validado con PC-EXT en Fase 3.

## Verificación end-to-end
Al cerrar cada fase se corre su chequeo (ping/tracert/servicios). Verificación global final:
matriz de accesos por segmento (permitidos y bloqueados) con capturas, más conectividad interna,
inter-VLAN, DMZ y acceso de Externos funcionando.

## Estado y próximos pasos
- [x] Plan y documentación (este archivo + `GLOSARIO.md`).
- [ ] **Fase 1** — arranca cuando el usuario lo indique (no se construye nada antes).
- [ ] Fases 2 a 5.
