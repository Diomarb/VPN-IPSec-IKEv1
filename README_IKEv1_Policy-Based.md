# VPN Site-to-Site IPSec IKEv1 — Basada en Políticas

**Asignatura:** Seguridad de Redes  

---

## Objetivo

Implementar una VPN Site-to-Site punto a punto utilizando el protocolo **IPSec con IKEv1**, configurada mediante el método basado en **políticas (policy-based)**. Este método utiliza listas de control de acceso (ACL) para definir qué tráfico debe ser encriptado y enviado a través del túnel, sin necesidad de interfaces de túnel virtuales.

---

## Topología
 <img width="784" height="644" alt="image" src="https://github.com/user-attachments/assets/54dd9715-2951-450b-8481-59f228be3ea3" />


### Direccionamiento IP

| Dispositivo | Interfaz | Dirección IP      | Subred              | Descripción          |
|-------------|----------|-------------------|---------------------|----------------------|
| R1          | e0/0     | 10.11.85.65/26    | 10.11.85.64/26      | WAN → ISP            |
| R1          | e0/1     | 10.11.85.1/26     | 10.11.85.0/26       | LAN → SW1            |
| ISP         | e0/0     | 10.11.85.66/26    | 10.11.85.64/26      | Hacia R1             |
| ISP         | e0/1     | 10.11.85.129/26   | 10.11.85.128/26     | Hacia R2             |
| R2          | e0/0     | 10.11.85.130/26   | 10.11.85.128/26     | WAN → ISP            |
| R2          | e0/1     | 10.11.85.193/26   | 10.11.85.192/26     | LAN → SW2            |
| PC1         | e0       | 10.11.85.2/26     | 10.11.85.0/26       | Gateway: 10.11.85.1  |
| PC2         | e0       | 10.11.85.194/26   | 10.11.85.192/26     | Gateway: 10.11.85.193|

---

## Parámetros IPSec IKEv1

### Fase 1 — IKE (ISAKMP)

| Parámetro       | Valor              |
|-----------------|--------------------|
| Política        | 10                 |
| Cifrado         | AES-256            |
| Hash            | SHA-256            |
| Autenticación   | Pre-shared key     |
| Grupo DH        | 14 (2048-bit)      |
| Lifetime        | 86400 segundos     |
| Clave PSK       | `ITLA2024`         |

### Fase 2 — IPSec (Transform Set)

| Parámetro          | Valor              |
|--------------------|--------------------|
| Nombre             | TS-IKEV1           |
| Protocolo          | ESP                |
| Cifrado            | AES-256            |
| Integridad         | SHA-256 (HMAC)     |
| Modo               | Tunnel             |

---

## Funcionamiento — Policy-based

En una VPN basada en políticas, el tráfico que debe ser encriptado se define mediante una **ACL extendida** (tráfico interesante). El router consulta esta ACL en cada paquete saliente; si el paquete coincide, se encripta y se envía al peer IPSec. No existe una interfaz de túnel virtual.

```
LAN R1 (10.11.85.0/26) ──► ACL match ──► ENCRYPT ──► [Internet/ISP] ──► DECRYPT ──► LAN R2 (10.11.85.192/26)
```

### Flujo de negociación IKEv1

1. **Fase 1 (Main Mode):** R1 y R2 negocian parámetros ISAKMP (cifrado, hash, autenticación, grupo DH) y establecen un canal seguro (SA de ISAKMP).
2. **Fase 2 (Quick Mode):** Dentro del canal de Fase 1, negocian los parámetros IPSec (Transform Set) y definen el tráfico a proteger (selector de tráfico desde la ACL). Se establecen las SAs IPSec bidireccionales.
3. **Transmisión:** Todo paquete que coincida con la ACL se encapsula en ESP y se envía encriptado al peer.

---

## Configuración paso a paso

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
!
crypto isakmp key ITLA2024 address 10.11.85.130
!
crypto ipsec transform-set TS-IKEV1 esp-aes 256 esp-sha256-hmac
 mode tunnel
!
ip access-list extended ACL-VPN-R1
 permit ip 10.11.85.0 0.0.0.63 10.11.85.192 0.0.0.63
!
crypto map CM-IKEV1 10 ipsec-isakmp
 set peer 10.11.85.130
 set transform-set TS-IKEV1
 match address ACL-VPN-R1
!
interface Ethernet0/0
 crypto map CM-IKEV1
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
!
crypto isakmp key ITLA2024 address 10.11.85.65
!
crypto ipsec transform-set TS-IKEV1 esp-aes 256 esp-sha256-hmac
 mode tunnel
!
ip access-list extended ACL-VPN-R2
 permit ip 10.11.85.192 0.0.0.63 10.11.85.0 0.0.0.63
!
crypto map CM-IKEV1 10 ipsec-isakmp
 set peer 10.11.85.65
 set transform-set TS-IKEV1
 match address ACL-VPN-R2
!
interface Ethernet0/0
 crypto map CM-IKEV1
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

### Comandos de verificación

```
! Ver SA de IKE Fase 1
show crypto isakmp sa

! Ver SA de IPSec Fase 2
show crypto ipsec sa

! Resultado esperado Fase 1 (QM_IDLE = establecido)
! dst            src            state          conn-id slot status
! 10.11.85.65    10.11.85.130   QM_IDLE        1001    0    ACTIVE

! Resultado esperado Fase 2
! local  ident (addr/mask/prot/port): (10.11.85.0/255.255.255.192/0/0)
! remote ident (addr/mask/prot/port): (10.11.85.192/255.255.255.192/0/0)
! #pkts encaps: X, #pkts encrypt: X, #pkts digest: X
! #pkts decaps: X, #pkts decrypt: X, #pkts verify: X
```

> **Nota:** El primer ping desde PC1 a PC2 puede perderse mientras se negocia la SA. Repetir el ping una segunda vez confirma que el túnel está activo.

---


