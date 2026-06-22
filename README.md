# VPN Site-to-Site IPSec IKEv1 — Túnel GRE (GRE over IPSec)

Link de video: https://youtu.be/I4qkQ60Q8pE
---

## Objetivo

Implementar una VPN Site-to-Site utilizando **GRE over IPSec con IKEv1**. En esta variante se combinan dos tecnologías:

- **GRE (Generic Routing Encapsulation):** crea un túnel lógico punto a punto que puede transportar cualquier protocolo de capa 3, incluyendo multicast/broadcast — lo que permite ejecutar protocolos de enrutamiento dinámico (OSPF, EIGRP) sobre el túnel.
- **IPSec IKEv1:** cifra y protege el tráfico GRE completo entre los peers, garantizando confidencialidad e integridad.

La diferencia clave con el route-based (VTI) es que aquí el `tunnel mode` es `gre ip` y IPSec se aplica en modo **transport** (no tunnel), ya que GRE ya provee el encapsulado de capa 3.

---

## Topología

<img width="784" height="644" alt="image" src="https://github.com/user-attachments/assets/266a74ac-52bf-4b6b-9ac3-e6eb34b2d84a" />

```

```

### Direccionamiento IP

| Dispositivo | Interfaz | Dirección IP       | Descripción             |
|-------------|----------|--------------------|-------------------------|
| R1          | e0/0     | 10.11.85.65/26     | WAN → ISP               |
| R1          | e0/1     | 10.11.85.1/26      | LAN → SW1               |
| R1          | Tunnel0  | 10.10.10.1/30      | GRE Tunnel hacia R2     |
| ISP         | e0/0     | 10.11.85.66/26     | Hacia R1                |
| ISP         | e0/1     | 10.11.85.129/26    | Hacia R2                |
| R2          | e0/0     | 10.11.85.130/26    | WAN → ISP               |
| R2          | e0/1     | 10.11.85.193/26    | LAN → SW2               |
| R2          | Tunnel0  | 10.10.10.2/30      | GRE Tunnel hacia R1     |
| PC1         | e0       | 10.11.85.2/26      | GW: 10.11.85.1          |
| PC2         | e0       | 10.11.85.194/26    | GW: 10.11.85.193        |

---

## Parámetros

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

### Fase 2 — IPSec

| Parámetro          | Valor                        |
|--------------------|------------------------------|
| Transform Set      | TS-GRE                       |
| Protocolo          | ESP                          |
| Cifrado            | AES-256                      |
| Integridad         | HMAC-SHA-256                 |
| **Modo**           | **Transport** ← diferencia clave |
| ACL tráfico        | `permit gre host X host Y`   |
| Crypto Map         | CM-GRE                       |

### Tunnel0 — GRE

| Parámetro          | R1                | R2                |
|--------------------|-------------------|-------------------|
| IP Tunnel          | 10.10.10.1/30     | 10.10.10.2/30     |
| Tunnel source      | Ethernet0/0       | Ethernet0/0       |
| Tunnel destination | 10.11.85.130      | 10.11.85.65       |
| **Tunnel mode**    | **gre ip**        | **gre ip**        |

---

## Funcionamiento

El flujo de encapsulación tiene dos etapas:

**Etapa 1 — GRE encapsula el tráfico LAN:**
```
PC1 (10.11.85.2) → PC2 (10.11.85.194)
R1: ruta hacia 10.11.85.192/26 → via Tunnel0
R1: GRE encapsula → nuevo paquete:
    IP outer src: 10.11.85.65  dst: 10.11.85.130
    GRE header
    IP inner  src: 10.11.85.2  dst: 10.11.85.194
```

**Etapa 2 — IPSec cifra el paquete GRE:**
```
ACL-GRE-VPN matchea: protocol GRE, src 10.11.85.65, dst 10.11.85.130
IPSec ESP (modo transport) cifra el payload GRE
Sale por e0/0 → ISP → R2
R2: IPSec descifra → GRE desencapsula → entrega a PC2
```

> **¿Por qué modo transport y no tunnel?**  
> GRE ya agrega un encabezado IP outer (10.11.85.65 → 10.11.85.130). Si IPSec usara modo tunnel, agregaría *otro* encabezado IP outer redundante. Modo transport evita esta doble encapsulación, reduciendo el overhead por paquete.

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
interface Tunnel0
 ip address 10.10.10.1 255.255.255.252
 tunnel source Ethernet0/0
 tunnel destination 10.11.85.130
 tunnel mode gre ip
 no shutdown
!
ip route 10.11.85.192 255.255.255.192 Tunnel0
!
crypto isakmp policy 10
 encr aes 256
 hash sha256
 authentication pre-share
 group 14
 lifetime 86400
crypto isakmp key ITLA2024 address 10.11.85.130
!
crypto ipsec transform-set TS-GRE esp-aes 256 esp-sha256-hmac
 mode transport
!
ip access-list extended ACL-GRE-VPN
 permit gre host 10.11.85.65 host 10.11.85.130
!
crypto map CM-GRE 10 ipsec-isakmp
 set peer 10.11.85.130
 set transform-set TS-GRE
 match address ACL-GRE-VPN
!
interface Ethernet0/0
 crypto map CM-GRE
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
interface Tunnel0
 ip address 10.10.10.2 255.255.255.252
 tunnel source Ethernet0/0
 tunnel destination 10.11.85.65
 tunnel mode gre ip
 no shutdown
!
ip route 10.11.85.0 255.255.255.192 Tunnel0
!
crypto isakmp policy 10
 encr aes 256
 hash sha256
 authentication pre-share
 group 14
 lifetime 86400
crypto isakmp key ITLA2024 address 10.11.85.65
!
crypto ipsec transform-set TS-GRE esp-aes 256 esp-sha256-hmac
 mode transport
!
ip access-list extended ACL-GRE-VPN
 permit gre host 10.11.85.130 host 10.11.85.65
!
crypto map CM-GRE 10 ipsec-isakmp
 set peer 10.11.85.65
 set transform-set TS-GRE
 match address ACL-GRE-VPN
!
interface Ethernet0/0
 crypto map CM-GRE
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
! Tunnel0 UP/UP con modo GRE
show interface Tunnel0

! Rutas (LAN remota vía Tunnel0)
show ip route

! IKE Fase 1 (QM_IDLE)
show crypto isakmp sa

! IPSec Fase 2 — selectores son "gre host X host Y"
show crypto ipsec sa

! ACL GRE matcheando
show ip access-lists ACL-GRE-VPN

! Ping y Trace

