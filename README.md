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

- `running-configs/` — R-USER.txt, FG-Server.conf, ISP.txt, SW-Usuarios.txt, PC-Usuario-ipsec.conf
- `scripts/setup-webserver.sh` — Apache con HTTPS, SSH y apagado ordenado
- `documentacion/` — informe completo en PDF
