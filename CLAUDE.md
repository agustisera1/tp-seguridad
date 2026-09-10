# CLAUDE.md — Guía para trabajar en este repo

Este repositorio es el **TP1 de Seguridad (UTN)**. El objetivo es construir en Cisco Packet
Tracer una red con firewall **ASA 5505**, salida a Internet, tres **VLANs** internas y una **DMZ**
con servidor web, aplicando **ACLs con mínimo privilegio**, y entregar un informe. El entregable
de red se llama **`RESOLUCION.pkt`** y se construye **desde cero**.

## Idioma
Responder siempre en **español** (rioplatense, natural). El usuario y toda la documentación están
en español.

## Documentos de referencia (leer antes de actuar)
- **`PLAN_FASES.md`** — plan maestro: fases, inventario, direccionamiento, matriz de ACLs, riesgos.
  Es la fuente de verdad del *qué* y el *en qué orden*.
- **`GLOSARIO.md`** — glosario de componentes, conceptos y CLI básico (material para el usuario).
- **`CONSIGNA.pdf`** — la consigna original de la cátedra (texto + imagen de la topología objetivo).

## Modo de desarrollo (IMPORTANTE)
El usuario es **principiante en Packet Tracer y en los CLI de Cisco/ASA**. El trabajo es
**guiado y asistido**, nunca autónomo:

- Entregar en cada paso: (a) **instrucciones concretas en la GUI de Packet Tracer** (qué
  dispositivo, qué cable, qué puerto) y (b) **configuración CLI lista para copiar/pegar**, comentada.
- **Explicar el "para qué"** de cada paso; no asumir que conoce términos (VLAN, trunk, ACL, NAT…).
- Trabajar por **fases y revisiones**: el usuario ejecuta en PT y pide validación; recién ahí se
  avanza a la fase siguiente. **No adelantar fases ni "construir" `RESOLUCION.pkt` por cuenta propia.**
- Cuando el usuario pida una revisión, validar contra el "estado al terminar" de la fase en
  `PLAN_FASES.md`.

## Disciplina de alcance (no entregar de más)
Es una cátedra universitaria: **ceñirse a la consigna**. Lo que la cátedra **no pide** queda como
**opcional** y se marca como tal:
- Hardening de equipos (SSH admin, claves cifradas, banner) → opcional.
- Navegación interna → Internet → opcional (solo hace falta la salida para el acceso de "Externos").

## Decisiones de diseño ya tomadas
- Se **descartó** `clase_2.pkt` (diseño incompatible). Se parte de cero con `RESOLUCION.pkt`.
- La **ASA 5505 hace todo**: perímetro (outside/dmz) **+ inter-VLAN** vía subinterfaces 802.1Q
  (trunk al SW-LAN). **Contingencia (Fase 2):** si la ASA 5505 de PT no soporta trunk/subif,
  pivotear a switch L3 / router interno; no cambia las fases 3–5.

## Topología y direccionamiento (resumen — detalle en PLAN_FASES.md)
- Internet → Router ISP `200.10.10.1/30` → ASA 5505 (`outside 200.10.10.2/30`, `inside` con VLANs,
  `dmz 192.168.40.1/24`) → SW-DMZ → WEB-SERVER `192.168.40.10/24`.
- VLAN 10 Admin `192.168.10.0/24` (gw .1) · VLAN 20 Sistemas `192.168.20.0/24` (gw .1) ·
  VLAN 30 Usuarios `192.168.30.0/24` (gw .1) · DMZ `192.168.40.0/24` · WAN `200.10.10.0/30`.
- Security-levels ASA: outside 0, dmz 50, inside/VLANs 100.

## ACLs (mínimo privilegio, destino = WEB-SERVER salvo aclaración)
- **Administración:** HTTPS, FTP.
- **Usuarios:** solo HTTPS.
- **Sistemas:** HTTPS, HTTP, FTP, SSH, ICMP + acceso a redes de Usuarios y Administración.
- **Externos (Internet):** HTTPS, FTP (subir archivos), contra la DMZ publicada por NAT.

## Notas técnicas
- Los archivos `.pkt` están **cifrados** (Twofish/EAX con clave `0x89`/IV `0x10`, doble ofuscación
  posicional y zlib). No son texto plano; para inspeccionar uno hay que descifrarlo.
- El Server-PT de Packet Tracer **no ofrece SSH nativo**: la regla "Sistemas → SSH" se resuelve en
  Fase 4 (permitir a nivel ACL o representar SSH contra un dispositivo de red en la DMZ).

## Entrega final
Informe **PDF** (con capturas de accesos permitidos/bloqueados) + `RESOLUCION.pkt`, comprimidos en
`TP1_SSI_ApellidoNombre.zip`.
