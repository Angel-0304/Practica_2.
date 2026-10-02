# Practica_2.
Infraestructura 3

# Infraestructura 3 — Web pública y SSH por VPN de acceso remoto

Ver video demostrativo: https://itlaedudo-my.sharepoint.com/:f:/g/personal/20242356_itla_edu_do/IgDN_aKp_D9ATL6FPikZjtAoATCooH1UvsXBNc7rNldfzxk?e=SyVIZZ

## Objetivos

- El usuario accede al servidor web sin necesidad de VPN.
- El usuario accede al servidor por SSH solo a través de la VPN.
- VPN de acceso remoto configurada en el FortiGate por GUI.
- Red del lado del usuario en un equipo Cisco.

## Diseño

La topología física es la misma de la Infraestructura 2, pero el router ya no participa en ninguna VPN. El FortiGate publica únicamente el puerto 443 mediante una VIP y ofrece una VPN IPsec de acceso remoto que el usuario levanta desde su equipo con strongSwan.

![Topología lógica](diagramas/topologia-logica.png)

| Segmento | Red / rango |
|---|---|
| Usuarios (VLAN 10) | 10.23.56.0/25 |
| Servidor | 10.23.56.128/28 (servidor .130) |
| Pool de la VPN | 10.23.56.200 – 10.23.56.210 |
| VIP web | 20.24.56.2:443 → 10.23.56.130:443 |

## FortiGate

**VIP con port forwarding solo para TCP 443.** Sin port forwarding la VIP publicaría todos los puertos, incluido el 22.

**VPN de acceso remoto** (VPN → IPsec Wizard → Remote Access, luego convertida a personalizada):

| Parámetro | Valor |
|---|---|
| Nombre | VPN-REMOTO |
| Autenticación | Pre-shared key + XAuth (grupo VPN-USUARIOS) |
| Phase 1 | IKEv1 agresivo, DES/MD5, DH 5 |
| Phase 2 | DES/MD5, PFS grupo 5 |
| Rango de clientes | 10.23.56.200 – 10.23.56.210 |
| Split tunnel | Activado (solo 10.23.56.130) |

**Políticas:**

| Nombre | Origen → Destino | Destino | Servicio | NAT |
|---|---|---|---|---|
| LAN-a-Internet | port2 → port1 | all | ALL | Sí |
| Internet-a-Web-HTTPS | port1 → port2 | VIP-WEB-HTTPS | HTTPS | No |
| VPN-a-Servidor-SSH | VPN-REMOTO → port2 | Web-Server | SSH, PING, TRACEROUTE | No |

## Cliente strongSwan

```
conn fortigate
    keyexchange=ikev1
    aggressive=yes
    ike=des-md5-modp1536!
    esp=des-md5-modp1536!
    left=%defaultroute
    leftid=@angel
    leftauth=psk
    leftauth2=xauth
    xauth_identity=angel
    leftsourceip=%config
    right=20.24.56.2
    rightid=%any
    rightauth=psk
    rightsubnet=10.23.56.130/32
    auto=add
```

## Verificación

**Sin VPN:**

```
curl -k https://20.24.56.2                  → muestra la página
ssh angel@20.24.56.2                        → Connection timed out
ssh angel@10.23.56.130                      → No route to host
```

**Con VPN (`ipsec up fortigate`):**

```
XAuth authentication of 'angel' (myself) successful
installing new virtual IP 10.23.56.200
CHILD_SA fortigate{1} established ... TS 10.23.56.200/32 === 10.23.56.130/32
```

```
ssh angel@10.23.56.130                      → entra al servidor
traceroute -n 10.23.56.130
 1  * * *
 2  10.23.56.130
```

**Evidencia en el servidor:**

| Servicio | Origen registrado | Camino |
|---|---|---|
| Web (Apache) | 20.24.23.2 | Internet, NAT del R-USER y VIP |
| SSH (auth.log) | 10.23.56.200 | Solo por la VPN |

## Problemas encontrados

| Problema | Causa | Solución |
|---|---|---|
| La web no cargaba (TLS sin respuesta) | Paquetes de 1500 bytes perdidos en el camino | `ip tcp adjust-mss 1360` en las interfaces del R-USER |
| "Maximum number of entries has been reached" | La licencia solo permite un portal SSL-VPN | Editar el portal existente |
| SSL-VPN imposible ("Cipher is (NONE)") | La licencia LENC solo ofrece cifrados que el OpenSSL actual no soporta | Cambiar a VPN IPsec de acceso remoto |
| No se podía quitar la interfaz de la SSL-VPN | La GUI la cambiaba a "any" | Dejarla en port4, sin cable ni IP |
| El PC sin ruta tras reiniciar el contenedor | Docker | Ruta temporal por eth0 y luego devolverla a eth1 |

## Archivos

## Running-configs

<details>
<summary><b>ISP</b></summary>

```
ISP#show running-config
Building configuration...

Current configuration : 1304 bytes
!
version 15.4
service timestamps debug datetime msec
service timestamps log datetime msec
no service password-encryption
!
hostname ISP
!
boot-start-marker
boot-end-marker
!
no aaa new-model
mmi polling-interval 60
no mmi auto-configure
no mmi pvc
mmi snmp-timeout 180
!
no ip domain lookup
ip cef
no ipv6 cef
!
multilink bundle-name authenticated
!
redundancy
!
interface Loopback0
 description Simula Internet
 ip address 8.8.8.8 255.255.255.255
!
interface Ethernet0/0
 description Enlace hacia R-USER e0/0
 ip address 20.24.23.1 255.255.255.252
!
interface Ethernet0/1
 description Enlace hacia FG-Server port1
 ip address 20.24.56.1 255.255.255.252
!
interface Ethernet0/2
 no ip address
 shutdown
!
interface Ethernet0/3
 no ip address
 shutdown
!
interface Ethernet1/0
 no ip address
 shutdown
!
interface Ethernet1/1
 no ip address
 shutdown
!
interface Ethernet1/2
 no ip address
 shutdown
!
interface Ethernet1/3
 no ip address
 shutdown
!
ip forward-protocol nd
!
no ip http server
no ip http secure-server
!
control-plane
!
banner motd ^CISP - Infraestructura 3 - Matricula 2024-2356^C
!
line con 0
 exec-timeout 0 0
 logging synchronous
line aux 0
line vty 0 4
 login
 transport input none
!
end
```

</details>

<details>
<summary><b>SW-Usuarios</b></summary>

```
SW-Usuarios#show running-config
Building configuration...

Current configuration : 1105 bytes
!
version 15.2
service timestamps debug datetime msec
service timestamps log datetime msec
no service password-encryption
service compress-config
!
hostname SW-Usuarios
!
boot-start-marker
boot-end-marker
!
no aaa new-model
!
no ip domain-lookup
ip cef
no ipv6 cef
!
spanning-tree mode rapid-pvst
spanning-tree extend system-id
!
vlan internal allocation policy ascending
!
interface Ethernet0/0
 description Trunk hacia R-USER e0/1
 switchport trunk allowed vlan 10
 switchport trunk encapsulation dot1q
 switchport mode trunk
!
interface Ethernet0/1
 description Access hacia PC-Usuario
 switchport access vlan 10
 switchport mode access
 spanning-tree portfast edge
!
interface Ethernet0/2
 shutdown
!
interface Ethernet0/3
 shutdown
!
ip forward-protocol nd
!
no ip http server
no ip http secure-server
!
control-plane
!
banner motd ^CSW-Usuarios - Infraestructura 3 - Matricula 2024-2356^C
!
line con 0
 exec-timeout 0 0
 logging synchronous
line aux 0
line vty 0 4
 login
!
end
```

</details>

<details>
<summary><b>R-USER</b></summary>

```
R-USER#show running-config
Building configuration...

Current configuration : 1625 bytes
!
version 15.4
service timestamps debug datetime msec
service timestamps log datetime msec
no service password-encryption
!
hostname R-USER
!
boot-start-marker
boot-end-marker
!
no aaa new-model
mmi polling-interval 60
no mmi auto-configure
no mmi pvc
mmi snmp-timeout 180
!
ip dhcp excluded-address 10.23.56.1 10.23.56.9
ip dhcp excluded-address 10.23.56.101 10.23.56.127
!
ip dhcp pool VLAN10-USUARIOS
 network 10.23.56.0 255.255.255.128
 default-router 10.23.56.1
 dns-server 8.8.8.8
!
no ip domain lookup
ip cef
no ipv6 cef
!
multilink bundle-name authenticated
!
redundancy
!
interface Ethernet0/0
 description WAN hacia ISP
 ip address 20.24.23.2 255.255.255.252
 ip nat outside
 ip virtual-reassembly in
 ip tcp adjust-mss 1360
!
interface Ethernet0/1
 description Trunk hacia SW-Usuarios
 no ip address
!
interface Ethernet0/1.10
 description VLAN 10 - Usuarios
 encapsulation dot1Q 10
 ip address 10.23.56.1 255.255.255.128
 ip nat inside
 ip virtual-reassembly in
 ip tcp adjust-mss 1360
!
interface Ethernet0/2
 no ip address
 shutdown
!
interface Ethernet0/3
 no ip address
 shutdown
!
ip forward-protocol nd
!
no ip http server
no ip http secure-server
ip nat inside source list NAT-LAN interface Ethernet0/0 overload
ip route 0.0.0.0 0.0.0.0 20.24.23.1
!
ip access-list extended NAT-LAN
 permit ip 10.23.56.0 0.0.0.127 any
!
control-plane
!
banner motd ^CR-USER - Infraestructura 3 - Matricula 2024-2356^C
!
line con 0
 exec-timeout 0 0
 logging synchronous
line aux 0
line vty 0 4
 login
 transport input none
!
end
```

</details>

<details>
<summary><b>FG-Server</b></summary>

> Extracto de la salida de `show` con las secciones configuradas en el laboratorio. Se omitieron los objetos y perfiles que FortiOS trae por defecto y las líneas `uuid`. Las contraseñas y la PSK se reemplazaron por marcadores.

```
FG-Server # show
#config-version=FGVMK6-6.4.0-FW-build1579-200330:opmode=1:vdom=0:user=admin
config system global
    set alias "FortiGate-VM64-KVM"
    set hostname "FG-Server"
    set timezone 04
end
config system interface
    edit "port1"
        set vdom "root"
        set ip 20.24.56.2 255.255.255.252
        set allowaccess ping
        set type physical
        set alias "WAN"
        set lldp-reception enable
        set role wan
        set snmp-index 1
    next
    edit "port2"
        set vdom "root"
        set ip 10.23.56.129 255.255.255.240
        set allowaccess ping
        set type physical
        set device-identification enable
        set lldp-transmission enable
        set role lan
        set snmp-index 2
    next
    edit "port3"
        set vdom "root"
        set mode dhcp
        set allowaccess ping https ssh http
        set type physical
        set snmp-index 3
        set defaultgw disable
    next
    edit "port4"
        set vdom "root"
        set type physical
        set snmp-index 4
    next
    edit "VPN-REMOTO"
        set vdom "root"
        set type tunnel
        set snmp-index 7
        set interface "port1"
    next
end
config firewall address
    edit "POOL-VPN"
        set type iprange
        set start-ip 10.23.56.200
        set end-ip 10.23.56.210
    next
    edit "Web-Server"
        set subnet 10.23.56.130 255.255.255.255
    next
    edit "VPN-REMOTO_range"
        set type iprange
        set comment "VPN: VPN-REMOTO (Created by VPN wizard)"
        set start-ip 10.23.56.200
        set end-ip 10.23.56.210
    next
end
config firewall addrgrp
    edit "VPN-REMOTO_split"
        set member "Web-Server"
        set comment "VPN: VPN-REMOTO (Created by VPN wizard)"
    next
end
config user local
    edit "angel"
        set type password
        set passwd <PASSWORD>
    next
end
config user group
    edit "VPN-USUARIOS"
        set member "angel"
    next
end
config vpn ssl web portal
    edit "tunnel-access"
        set tunnel-mode enable
        set ipv6-tunnel-mode enable
        set ip-pools "POOL-VPN"
        set ipv6-pools "SSLVPN_TUNNEL_IPv6_ADDR1"
    next
end
config vpn ssl settings
    set ssl-min-proto-ver tls1-0
    set servercert "Fortinet_Factory"
    set tunnel-ip-pools "SSLVPN_TUNNEL_ADDR1"
    set source-interface "port4"
    set source-address "all"
    set source-address6 "all"
    set default-portal "tunnel-access"
    config authentication-rule
        edit 1
            set groups "VPN-USUARIOS"
            set portal "tunnel-access"
        next
    end
end
config vpn ipsec phase1-interface
    edit "VPN-REMOTO"
        set type dynamic
        set interface "port1"
        set mode aggressive
        set peertype any
        set net-device disable
        set mode-cfg enable
        set proposal des-md5
        set comments "VPN: VPN-REMOTO (Created by VPN wizard)"
        set dhgrp 5
        set xauthtype auto
        set authusrgrp "VPN-USUARIOS"
        set ipv4-start-ip 10.23.56.200
        set ipv4-end-ip 10.23.56.210
        set dns-mode auto
        set ipv4-split-include "VPN-REMOTO_split"
        set save-password enable
        set psksecret <PRE-SHARED-KEY>
    next
end
config vpn ipsec phase2-interface
    edit "VPN-REMOTO"
        set phase1name "VPN-REMOTO"
        set proposal des-md5
        set dhgrp 5
        set comments "VPN: VPN-REMOTO (Created by VPN wizard)"
    next
end
config firewall vip
    edit "VIP-WEB-HTTPS"
        set extip 20.24.56.2
        set extintf "port1"
        set portforward enable
        set mappedip "10.23.56.130"
        set extport 443
        set mappedport 443
    next
end
config firewall policy
    edit 1
        set name "LAN-a-Internet"
        set srcintf "port2"
        set dstintf "port1"
        set srcaddr "all"
        set dstaddr "all"
        set action accept
        set schedule "always"
        set service "ALL"
        set nat enable
    next
    edit 2
        set name "Internet-a-Web-HTTPS"
        set srcintf "port1"
        set dstintf "port2"
        set srcaddr "all"
        set dstaddr "VIP-WEB-HTTPS"
        set action accept
        set schedule "always"
        set service "HTTPS"
    next
    edit 3
        set name "VPN-a-Servidor-SSH"
        set srcintf "VPN-REMOTO"
        set dstintf "port2"
        set srcaddr "VPN-REMOTO_range"
        set dstaddr "Web-Server"
        set action accept
        set schedule "always"
        set service "PING" "SSH" "TRACEROUTE"
        set comments "VPN: VPN-REMOTO (Created by VPN wizard)"
    next
end
config router static
    edit 1
        set gateway 20.24.56.1
        set device "port1"
    next
end
```

> En la Phase 1 y Phase 2 no aparecen `ike-version 1`, `keylife 86400`, `pfs enable` ni `keylifeseconds 43200` porque son los valores por defecto de FortiOS 6.4. La configuración de SSL-VPN quedó de la primera prueba, descartada (ver sección de problemas): escucha en port4, que no tiene cable ni IP.

</details>



- `documentacion/` — informe completo en PDF
