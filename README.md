# TP1 — Seguridad (UTN)

Trabajo práctico de la materia **Seguridad**. Consiste en diseñar y configurar en **Cisco Packet
Tracer** una red corporativa con equipo perimetral, salida a **Internet**, tres **VLANs** internas
(Administración, Sistemas, Usuarios) y una **DMZ** con un servidor web, aplicando **ACLs con
mínimo privilegio**.

El entregable de red se construye **desde cero** en el archivo **`RESOLUCION.pkt`**.

## Topología actual

```
        INTERNET (nube)
            |
        Router ISP  200.10.10.1/30
            | (outside 200.10.10.2/30)
      Router 4331 ───────── dmz 192.168.40.1/24 ── SW-DMZ ── WEB-SERVER 192.168.40.10/24
            | (subinterfaces 802.1Q, trunk a SW-LAN)
          SW-LAN
        /   |   \
   VLAN10 VLAN20 VLAN30
   ADMIN  SISTEMAS USUARIOS
```

*(El equipo perimetral es un Router Cisco 4331, no la ASA 5505 de la imagen original de la
consigna — límite de licencia, ver `FIREWALL_LICENSE_ISSUE.md`.)*

| Segmento | VLAN | Red / Máscara | Gateway |
|---|---|---|---|
| Administración | 10 | 192.168.10.0 /24 | 192.168.10.1 |
| Sistemas | 20 | 192.168.20.0 /24 | 192.168.20.1 |
| Usuarios | 30 | 192.168.30.0 /24 | 192.168.30.1 |
| DMZ | — | 192.168.40.0 /24 | 192.168.40.1 |
| WAN | — | 200.10.10.0 /30 | ISP .1 / Router 4331 .2 |

## Consignas del TP (resumen)
1. Armar la red en Packet Tracer como en la imagen de la consigna.
2. Configurar IP, máscara y default gateway en todos los equipos (incluidos los de red).
3. Configurar la comunicación entre VLANs.
4. Crear ACLs con mínimo privilegio: Administración (HTTPS+FTP), Usuarios (solo HTTPS),
   Sistemas (HTTPS+HTTP+FTP+SSH+ICMP + acceso a redes de Usuarios y Admin), Externos (HTTPS+FTP).
5. Analizar la topología y proponer dispositivos de seguridad adicionales (justificado).

**Entrega:** informe PDF (con capturas de accesos permitidos/bloqueados) + `RESOLUCION.pkt`,
comprimidos en `TP1_SSI_ApellidoNombre.zip`.

## Cómo está organizado el desarrollo
El trabajo avanza en **6 fases graduales**. Plan completo en **[`PLAN_FASES.md`](PLAN_FASES.md)**;
pasos ejecutados (GUI+CLI) en **[`EJECUCION.md`](EJECUCION.md)**.

1. ✅ **Fase 1** — Topología física + direccionamiento (consignas 1 y 2). Completa.
2. ✅ **Fase 2** — Migración a Router 4331 + inter-VLAN (consigna 3). Completa.
3. ✅ **Fase 3** — Salida a Internet + publicación de DMZ por NAT. Completa.
4. ✅ **Fase 4** — ACLs con mínimo privilegio (consigna 4). Completa.
5. ✅ **Fase 5** — Análisis de dispositivos de seguridad adicionales (consigna 5). Completa —
   ver `ANALISIS_SEGURIDAD.md`.
6. ▶️ **Fase 6** — Pruebas finales, evidencia y entrega. Próxima a hacer.

## Archivos del repositorio
| Archivo | Qué es |
|---|---|
| `CONSIGNA.pdf` | Consigna original de la cátedra. |
| `PLAN_FASES.md` | Plan maestro: fases, inventario, direccionamiento, ACLs, riesgos. |
| `EJECUCION.md` | Bitácora paso a paso (GUI+CLI) de lo ya configurado, fase por fase, con checklist. |
| `GLOSARIO.md` | Glosario de componentes, conceptos y CLI básico (apoyo para el armado). |
| `FIREWALL_LICENSE_ISSUE.md` | Por qué se migró de ASA 5505 a Router 4331. |
| `ANALISIS_SEGURIDAD.md` | Análisis de consigna 5 (dispositivos de seguridad adicionales propuestos). |
| `CLAUDE.md` | Guía de trabajo para el asistente (modo de desarrollo y decisiones). |
| `RESOLUCION.pkt` | La red de Packet Tracer. Fases 1-4 construidas y verificadas; falta la evidencia de Fase 6. |

## Nota sobre los archivos `.pkt`
Los archivos de Packet Tracer están **cifrados** (no son texto plano), por lo que se abren
únicamente con Packet Tracer.
