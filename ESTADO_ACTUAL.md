# TP1 — Seguridad · Estado actual de `clase_2.pkt`

> Documento generado a partir del descifrado y análisis del archivo `clase_2.pkt`
> (Cisco Packet Tracer). Refleja el **estado tal como está guardado** en el `.pkt`,
> no un diseño objetivo.

- **Archivo:** `clase_2.pkt`
- **Versión de Packet Tracer:** 9.0.1
- **Dispositivos:** 10 (2 routers, 2 switches, 1 servidor, 4 PCs, 1 Power Distribution Device)

---

## 1. Topología

```
   VLAN 10                                                      red 172.16.x (sin ruteo)
   PC0 192.168.10.10 ─┐                                    ┌─ PC2 172.16.10.10
                      ├─ S1 ═══ R-Rosario ═══════ Router ═══┤
   PC1 192.168.20.10 ─┘ (Switch0)   (Router0)  G0/0 (Router1) (Switch1)
   VLAN 20                                                   ├─ PC3 172.16.20.10
                                                             └─ Server0 190.1.3.10 (DNS)
```

### Cableado (según el `.pkt`)

| Extremo A | Interfaz A | Tipo de cable | Extremo B | Interfaz B |
|---|---|---|---|---|
| PC0 | Fa0 | CrossOver | S1 (Switch0) | Fa0/1 |
| PC1 | Fa0 | CrossOver | S1 (Switch0) | Fa0/2 |
| PC2 | Fa0 | CrossOver | Switch (Switch1) | Fa0/1 |
| PC3 | Fa0 | CrossOver | Switch (Switch1) | Fa0/2 |
| S1 (Switch0) | G0/1 | CrossOver | R-Rosario (Router0) | G0/1 |
| Switch (Switch1) | G0/1 | CrossOver | Router (Router1) | G0/1 |
| R-Rosario (Router0) | G0/0 | CrossOver | Router (Router1) | G0/0 |
| Switch (Switch1) | Fa0/3 | StraightThrough | Server0 | Fa0 |

---

## 2. Inventario de dispositivos

| Dispositivo | Hostname | Modelo | Estado de config |
|---|---|---|---|
| Router0 | **R-Rosario** | Cisco 1941 | Configurado (router-on-a-stick + OSPF + SSH) |
| Router1 | **Router** | Cisco 1941 | **Sin configurar** (interfaces en `shutdown`, sin IP) |
| Switch0 | **S1** | 2960-24TT | Configurado (VLANs, trunk, SSH, puertos no usados en `shutdown`) |
| Switch1 | **Switch** | 2960-24TT | **Sin configurar** (todo por defecto) |
| Server0 | — | Server-PT | Servicio DNS activo |
| PC0 | — | PC-PT | Host VLAN 10 |
| PC1 | — | PC-PT | Host VLAN 20 |
| PC2 | — | PC-PT | Host red 172.16.10.0 |
| PC3 | — | PC-PT | Host red 172.16.20.0 |
| Power Distribution Device0 | — | PDU | — |

---

## 3. Direccionamiento IP

### Hosts finales

| Host | IP | Máscara | Gateway | Notas |
|---|---|---|---|---|
| PC0 | 192.168.10.10 | /24 | 192.168.10.254 | Gateway = subinterfaz `G0/1.10` de R-Rosario ✔ |
| PC1 | 192.168.20.10 | /24 | 192.168.20.254 | Gateway = subinterfaz `G0/1.20` de R-Rosario ✔ |
| PC2 | 172.16.10.10 | /24 | 172.16.10.254 | Gateway **inexistente** (nadie configurado) ✖ |
| PC3 | 172.16.20.10 | /24 | 172.16.20.254 | Gateway **inexistente** ✖ |
| Server0 | 190.1.3.10 | /24 | 190.1.3.254 | Gateway **inexistente**; corre servicio DNS ✖ |

### Interfaces de router

| Router | Interfaz | IP | Detalle |
|---|---|---|---|
| R-Rosario | G0/0 | 200.1.1.1 /26 | Enlace inter-router |
| R-Rosario | G0/1 | (sin IP) | Físico del trunk 802.1Q |
| R-Rosario | G0/1.10 | 192.168.10.254 /24 | `encapsulation dot1Q 10` |
| R-Rosario | G0/1.20 | 192.168.20.254 /24 | `encapsulation dot1Q 20` |
| Router (Router1) | G0/0 | — | `shutdown`, sin IP |
| Router (Router1) | G0/1 | — | `shutdown`, sin IP |

---

## 4. VLANs y switching

### S1 (Switch0) — configurado

| Interfaz | Modo | VLAN / detalle |
|---|---|---|
| Fa0/1 | access | VLAN 10 |
| Fa0/2 | access | VLAN 20 |
| G0/1 | **trunk** | allowed vlan 10,20 |
| Fa0/3 – Fa0/24 | — | `shutdown` (puertos no usados apagados) ✔ |

### Switch (Switch1) — sin configurar

- Todas las interfaces en su estado por defecto (VLAN 1, sin `shutdown`, sin trunk).

---

## 5. Ruteo

- **R-Rosario:** OSPF proceso 1, área 0, anunciando:
  - `200.1.1.0 0.0.0.63` (enlace inter-router)
  - `192.168.10.0 0.0.0.255`
  - `192.168.20.0 0.0.0.255`
- **Router (Router1):** sin ruteo, sin IPs → la mitad derecha de la topología (172.16.x, Server 190.1.3.10) **no tiene conectividad**.

---

## 6. Hallazgos de seguridad (estado actual)

> Este `.pkt` parece un TP de Seguridad. Los puntos siguientes son el estado
> **tal como está**, útiles como línea de base antes de endurecer la config.

| # | Hallazgo | Dónde | Severidad |
|---|---|---|---|
| 1 | `no service password-encryption` → contraseñas de línea en texto plano | R-Rosario, S1 | Media |
| 2 | Password de consola débil y en claro: `line con 0 / password utn` | R-Rosario, S1 | Media |
| 3 | `enable secret` y `username admin secret` usan el **mismo hash MD5** (tipo 5, `$1$mERr$...`), crackeable, y se **reutiliza** entre R-Rosario y S1 | R-Rosario, S1 | Alta |
| 4 | `line vty` con `login` sin contraseña / `transport` por defecto en dispositivos sin configurar | Router1, Switch1 | Alta |
| 5 | Sin ACLs, sin `port-security`, sin control de acceso en puertos de acceso | Todos | Media |
| 6 | Mitad derecha de la red sin gateway/ruteo: PC2, PC3 y Server0 aislados | Router1, Switch1 | Funcional/Disponibilidad |
| 7 | Server0 con DNS expuesto pero sin ruteo ni protección | Server0 | Baja (por aislamiento) |

**Buenas prácticas ya presentes:** SSH habilitado (`transport input ssh`), `login local`, `ip domain-name utn.com`, banner MOTD de advertencia, puertos no usados de S1 en `shutdown`.

---

## 7. Cómo se leyó el archivo

El `.pkt` **no es texto plano**: Packet Tracer aplica, en este orden inverso al abrir:

1. Ofuscación posicional (`b[i] = a[len-1-i] ^ (len - i*len)`).
2. Cifrado **Twofish** en modo **EAX** (clave = `0x89`×16, IV = `0x10`×16).
3. Segunda ofuscación posicional (`b[i] = a[i] ^ (len - i)`).
4. Descompresión **zlib** (con los primeros 4 bytes = tamaño original).

El XML resultante (~1 MB) contiene toda la topología y las `running-config`.

---

## 8. Pendientes / próximos pasos sugeridos

- [ ] Configurar Router1 (`Router`) y Switch1 (`Switch`): VLANs, trunk, IPs, ruteo.
- [ ] Dar gateway válido y conectividad a PC2, PC3 y Server0.
- [ ] Endurecer credenciales: `service password-encryption`, secretos distintos por dispositivo y por usuario.
- [ ] Definir política de acceso (ACLs, `port-security`).
- [ ] Documentar el objetivo del TP para contrastar "estado actual" vs "estado deseado".
