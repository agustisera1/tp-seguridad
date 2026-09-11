# TP1 — Seguridad (UTN)

Trabajo práctico de la materia **Seguridad**. Consiste en diseñar y configurar en **Cisco Packet
Tracer** una red corporativa con **firewall ASA 5505**, salida a **Internet**, tres **VLANs**
internas (Administración, Sistemas, Usuarios) y una **DMZ** con un servidor web, aplicando
**listas de control de acceso (ACLs) con criterio de mínimo privilegio**.

El entregable de red se construye **desde cero** en el archivo **`RESOLUCION.pkt`**.

## Topología objetivo

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
```

| Segmento | VLAN | Red / Máscara | Gateway |
|---|---|---|---|
| Administración | 10 | 192.168.10.0 /24 | 192.168.10.1 |
| Sistemas | 20 | 192.168.20.0 /24 | 192.168.20.1 |
| Usuarios | 30 | 192.168.30.0 /24 | 192.168.30.1 |
| DMZ | — | 192.168.40.0 /24 | 192.168.40.1 |
| WAN | — | 200.10.10.0 /30 | ISP .1 / ASA .2 |

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
El trabajo avanza en **5 fases graduales**, cada una con un estado verificable antes de pasar a la
siguiente. Plan completo en **[`PLAN_FASES.md`](PLAN_FASES.md)**; pasos ya ejecutados (GUI+CLI)
en **[`EJECUCION.md`](EJECUCION.md)**.

1. ✅ **Fase 1** — Topología física + direccionamiento (consignas 1 y 2). **Completa.**
2. ▶️ **Fase 2** — Inter-VLAN (consigna 3). **Próxima a hacer.**
3. **Fase 3** — Salida a Internet + publicación de DMZ por NAT.
4. **Fase 4** — ACLs con mínimo privilegio (consigna 4).
5. **Fase 5** — Análisis, verificación final y entrega (consigna 5).

> **Para quien siga con Fase 2:** la ASA 5505 de este simulador tiene licencia **Base** (máximo ~2
> interfaces `nameif` libres + 1 restringida). Con `outside`+`dmz` ya ocupados, es muy probable que
> haga falta pivotear el inter-VLAN a un switch L3 / router interno en vez de subinterfaces en la
> ASA. Detalle en `PLAN_FASES.md` (checkpoint de contingencia) y `EJECUCION.md` (Fase 1, Tarea 4).

## Archivos del repositorio
| Archivo | Qué es |
|---|---|
| `CONSIGNA.pdf` | Consigna original de la cátedra. |
| `PLAN_FASES.md` | Plan maestro: fases, inventario, direccionamiento, ACLs, riesgos. |
| `EJECUCION.md` | Bitácora paso a paso (GUI+CLI) de lo ya configurado, fase por fase, con checklist. |
| `GLOSARIO.md` | Glosario de componentes, conceptos y CLI básico (apoyo para el armado). |
| `CLAUDE.md` | Guía de trabajo para el asistente (modo de desarrollo y decisiones). |
| `RESOLUCION.pkt` | La red de Packet Tracer. Fase 1 ya construida; se sigue completando por fases. |

## Nota sobre los archivos `.pkt`
Los archivos de Packet Tracer están **cifrados** (no son texto plano), por lo que se abren
únicamente con Packet Tracer.
