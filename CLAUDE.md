# CLAUDE.md — Guía para trabajar en este repo

Este repositorio es el **TP1 de Seguridad (UTN)**. El objetivo es construir en Cisco Packet
Tracer una red con un equipo perimetral haciendo de firewall (ver `FIREWALL_LICENSE_ISSUE.md` para
por qué es un **Router 4331** y no la ASA 5505 de la imagen original), salida a Internet, tres
**VLANs** internas y una **DMZ** con servidor web, aplicando **ACLs con mínimo privilegio**, y
entregar un informe. El entregable de red se llama **`RESOLUCION.pkt`** y se construye
**desde cero**.

## Idioma
Responder siempre en **español** (rioplatense, natural). El usuario y toda la documentación están
en español.

## Documentos de referencia (leer antes de actuar)
- **`PLAN_FASES.md`** — plan maestro: fases, inventario, direccionamiento, matriz de ACLs, riesgos.
  Es la fuente de verdad del *qué* y el *en qué orden*.
- **`GLOSARIO.md`** — glosario de componentes, conceptos y CLI básico (material para el usuario).
  Todo término técnico nuevo que aparezca en cualquier documento **tiene que estar acá**.
- **`FIREWALL_LICENSE_ISSUE.md`** — por qué se migró de ASA 5505 a Router 4331 (límite de
  licencia) + el tradeoff router-vs-firewall dedicado.
- **`CONSIGNA.pdf`** — la consigna original de la cátedra (texto + imagen de la topología objetivo).

## Modo de desarrollo (IMPORTANTE)
El usuario es **principiante en Packet Tracer y en los CLI de Cisco/ASA/IOS — no sabe nada del
tema**. El trabajo es **guiado y asistido**, nunca autónomo:

- Entregar en cada paso: (a) **instrucciones concretas en la GUI de Packet Tracer** (qué
  dispositivo, qué cable, qué puerto) y (b) **configuración CLI lista para copiar/pegar**, comentada.
- **Todo texto del proyecto va "en criollo"**: lenguaje simple, con analogías si hace falta, dando
  por hecho que el usuario no sabe nada de redes/seguridad. Nada de jerga sin explicar en el
  momento en que aparece.
- **Todo término técnico usado en cualquier documento debe estar en `GLOSARIO.md`.** Si se
  introduce un concepto nuevo (una fase nueva, un dispositivo nuevo, etc.), se agrega al glosario
  en el mismo momento, no después.
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
- **El Router 4331 hace todo** (perímetro + inter-VLAN), no la ASA 5505 de la imagen original de
  la consigna: la ASA con licencia Base solo permite 3 interfaces con nombre en total y hacen
  falta 5. La consigna permite el reemplazo ("1 Router Cisco 4331 / Firewall ASA"). Motivo
  completo y en criollo: `FIREWALL_LICENSE_ISSUE.md`. Inter-VLAN vía subinterfaces 802.1Q
  (trunk al SW-LAN), igual que se hubiera hecho en la ASA.

## Topología y direccionamiento (resumen — detalle en PLAN_FASES.md)
- Internet → Router ISP `200.10.10.1/30` → Router 4331 (`outside 200.10.10.2/30`, subinterfaces
  por VLAN, `dmz 192.168.40.1/24`) → SW-DMZ → WEB-SERVER `192.168.40.10/24`.
- VLAN 10 Admin `192.168.10.0/24` (gw .1) · VLAN 20 Sistemas `192.168.20.0/24` (gw .1) ·
  VLAN 30 Usuarios `192.168.30.0/24` (gw .1) · DMZ `192.168.40.0/24` · WAN `200.10.10.0/30`.
- El Router 4331 (IOS) no tiene *security-levels* como la ASA: el control de acceso entre
  segmentos se hace 100% con ACLs extendidas por interfaz (Fase 4).

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
