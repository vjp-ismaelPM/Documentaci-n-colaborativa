# Diseño de Red

## Diagrama (texto)

Internet
   |
[Firewall]
   |
[Core Switch]
 /    |    \
VLAN10 VLAN20 VLAN30
Admin   TI   Invitados

## Equipos
- Firewall: FortiGate 60F
- Core Switch: Cisco Catalyst 9200
- APs: Ubiquiti UniFi

## Direccionamiento
| VLAN | Nombre    | Red             |
|------|-----------|-----------------|
| 10   | Admin     | 192.168.10.1/24 |
| 20   | TI        | 192.168.20.2/24 |
| 30   | Invitados | 192.168.30.3/24 |

| Apache     | 2.4.59 | Servidor web (versión actualizada) |
| MySQL      | 8.0    | Base de datos                      |
| Certbot    | 2.9    | SSL/TLS automático (añadido)       |
Certbot para SSL