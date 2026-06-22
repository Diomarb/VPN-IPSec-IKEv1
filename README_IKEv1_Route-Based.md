# VPN Site-to-Site IPSec IKEv1 — Basada en Enrutamiento (VTI)

**Estudiante:** Euni  
**Matrícula:** 2024-1185  
**Institución:** Instituto Tecnológico de las Américas (ITLA)  
**Asignatura:** Seguridad de Redes  

---

## Objetivo

Implementar una VPN Site-to-Site punto a punto utilizando **IPSec con IKEv1** mediante el método **basado en enrutamiento (route-based)**. A diferencia del método policy-based, aquí se crea una **interfaz virtual de túnel (VTI — Virtual Tunnel Interface)** llamada `Tunnel0`. El tráfico cifrado se determina por la tabla de enrutamiento: cualquier paquete que el router envíe a través de `Tunnel0` es encapsulado y cifrado automáticamente por IPSec.

---

## Topología

```
[PC1]──[SW1]──[R1]──────[ISP]──────[R2]──[SW2]──[PC2]
              e0/1  e0/0       e0/1  e0/0  e0/1
                    └──────────────────┘
                      Tunnel0 ↔ Tunnel0
                    10.10.10.1   10.10.10.2
```

### Direccionamiento IP

| Dispositivo | Interfaz | Dirección IP       | Descripción             |
|-------------|----------|--------------------|-------------------------|
| R1          | e0/0     | 10.11.85.65/26     | WAN → ISP               |
| R1          | e0/1     | 10.11.85.1/26      | LAN → SW1               |
| R1          | Tunnel0  | 10.10.10.1/30      | VTI hacia R2            |
| ISP         | e0/0     | 10.11.85.66/26     | Hacia R1                |
| ISP         | e0/1     | 10.11.85.129/26    | Hacia R2                |
| R2          | e0/0     | 10.11.85.130/26    | WAN → ISP               |
| R2          | e0/1     | 10.11.85.193/26    | LAN → SW2               |
| R2          | Tunnel0  | 10.10.10.2/30      | VTI hacia R1            |
| PC1         | e0       | 10.11.85.2/26      | GW: 10.11.85.1          |
| PC2         | e0       | 10.11.85.194/26    | GW: 10.11.85.193        |

---

## Parámetros IPSec IKEv1

### Fase 1 — IKE / ISAKMP

| Parámetro          | Valor                        |
|--------------------|------------------------------|
| Política           | 10                           |
| Cifrado            | AES-256                      |
| Hash               | SHA-256                      |
| Autenticación      | Pre-Shared Key               |
| Grupo DH           | 14 (2048-bit)                |
| Lifetime           | 86400 segundos               |
| Clave PSK          | `ITLA2024`                   |

### Fase 2 — IPSec / Transform Set

| Parámetro          | Valor                        |
|--------------------|------------------------------|
| Nombre TS          | TS-IKEV1-VTI                 |
| Protocolo          | ESP                          |
| Cifrado            | AES-256                      |
| Integridad         | HMAC-SHA-256                 |
| Modo               | Tunnel                       |
| IPSec Profile      | PROFILE-VTI                  |

### Tunnel0

| Parámetro          | R1                | R2                |
|--------------------|-------------------|-------------------|
| IP Tunnel          | 10.10.10.1/30     | 10.10.10.2/30     |
| Tunnel source      | Ethernet0/0       | Ethernet0/0       |
| Tunnel destination | 10.11.85.130      | 10.11.85.65       |
| Tunnel mode        | ipsec ipv4        | ipsec ipv4        |
| Tunnel protection  | PROFILE-VTI       | PROFILE-VTI       |

---

## Funcionamiento — Route-Based VPN

En una VPN basada en enrutamiento, la interfaz `Tunnel0` actúa como un enlace punto a punto virtual. IPSec protege automáticamente **todo el tráfico** que el router decide enviar por esa interfaz, lo que se controla simplemente con rutas estáticas (o protocolos de enrutamiento dinámico como OSPF o EIGRP).

```
PC1 → R1 consulta tabla de rutas:
  Destino 10.11.85.192/26 → via Tunnel0
    └─ Tunnel0 tiene tunnel protection ipsec profile
       └─ IPSec cifra el paquete con AES-256
          └─ Sale por e0/0 hacia ISP → R2
             R2 descifra → entrega a PC2
```

**Ventaja principal sobre policy-based:** no se necesita ACL. Si se añade una nueva subred, basta agregar una ruta hacia `Tunnel0`, sin tocar la configuración IPSec.

---

## Configuración

### ISP

```
hostname ISP
!
interface Ethernet0/0
 ip address 10.11.85.66 255.255.255.192
 no shutdown
!
interface Ethernet0/1
 ip address 10.11.85.129 255.255.255.192
 no shutdown
```

### R1

```
hostname R1
!
interface Ethernet0/0
 ip address 10.11.85.65 255.255.255.192
 no shutdown
!
interface Ethernet0/1
 ip address 10.11.85.1 255.255.255.192
 no shutdown
!
ip route 0.0.0.0 0.0.0.0 10.11.85.66
!
crypto isakmp policy 10
 encr aes 256
 hash sha256
 authentication pre-share
 group 14
 lifetime 86400
crypto isakmp key ITLA2024 address 10.11.85.130
!
crypto ipsec transform-set TS-IKEV1-VTI esp-aes 256 esp-sha256-hmac
 mode tunnel
!
crypto ipsec profile PROFILE-VTI
 set transform-set TS-IKEV1-VTI
!
interface Tunnel0
 ip address 10.10.10.1 255.255.255.252
 tunnel source Ethernet0/0
 tunnel destination 10.11.85.130
 tunnel mode ipsec ipv4
 tunnel protection ipsec profile PROFILE-VTI
 no shutdown
!
ip route 10.11.85.192 255.255.255.192 Tunnel0
```

### R2

```
hostname R2
!
interface Ethernet0/0
 ip address 10.11.85.130 255.255.255.192
 no shutdown
!
interface Ethernet0/1
 ip address 10.11.85.193 255.255.255.192
 no shutdown
!
ip route 0.0.0.0 0.0.0.0 10.11.85.129
!
crypto isakmp policy 10
 encr aes 256
 hash sha256
 authentication pre-share
 group 14
 lifetime 86400
crypto isakmp key ITLA2024 address 10.11.85.65
!
crypto ipsec transform-set TS-IKEV1-VTI esp-aes 256 esp-sha256-hmac
 mode tunnel
!
crypto ipsec profile PROFILE-VTI
 set transform-set TS-IKEV1-VTI
!
interface Tunnel0
 ip address 10.10.10.2 255.255.255.252
 tunnel source Ethernet0/0
 tunnel destination 10.11.85.65
 tunnel mode ipsec ipv4
 tunnel protection ipsec profile PROFILE-VTI
 no shutdown
!
ip route 10.11.85.0 255.255.255.192 Tunnel0
```

### VPCS

```
! PC1
ip 10.11.85.2 255.255.255.192 10.11.85.1
save

! PC2
ip 10.11.85.194 255.255.255.192 10.11.85.193
save
```

---

## Verificación

```
! Ver interfaz Tunnel0 (debe estar UP/UP)
show interface Tunnel0

! Ver tabla de rutas (LAN remota vía Tunnel0)
show ip route

! IKE Fase 1 (debe mostrar QM_IDLE)
show crypto isakmp sa

! IPSec Fase 2 + contadores de paquetes
show crypto ipsec sa

! Profile aplicado
show crypto ipsec profile

! Hora y fecha para el video
show clock
```

**Salida esperada de `show ip route` en R1:**
```
S    10.11.85.192/26 is directly connected, Tunnel0
```

**Salida esperada de `show interface Tunnel0`:**
```
Tunnel0 is up, line protocol is up
  Hardware is Tunnel
  Internet address is 10.10.10.1/30
  Tunnel source 10.11.85.65 (Ethernet0/0), destination 10.11.85.130
  Tunnel protocol/transport IPSEC/IP
```

---

## Diferencia clave vs Policy-Based

| Característica         | Policy-Based              | Route-Based (este lab)          |
|------------------------|---------------------------|---------------------------------|
| Selección de tráfico   | ACL extendida             | Tabla de enrutamiento           |
| Interfaz de túnel      | No existe                 | Tunnel0 (VTI)                   |
| Crypto Map             | Requerido en e0/0         | **No se usa**                   |
| IPSec Profile          | No se usa                 | Requerido en Tunnel0            |
| Enrutamiento dinámico  | No soportado directo      | **Soportado** (OSPF, EIGRP)     |
| Agregar nueva subred   | Modificar ACL + crypto map| Solo agregar ruta a Tunnel0     |

---

## Contramédidas

| Amenaza                   | Contramédida                                           |
|---------------------------|--------------------------------------------------------|
| Interceptación            | AES-256 cifra todo el tráfico en el VTI                |
| Modificación de paquetes  | HMAC-SHA-256 verifica integridad                       |
| Replay attacks            | Anti-replay habilitado por defecto en ESP              |
| Acceso no autorizado      | PSK + DH Group 14 para establecer la SA               |
| Fallo de túnel visible    | `show interface Tunnel0` → down si IPSec falla        |

---

*Documento generado para fines académicos — ITLA 2024-1185*
