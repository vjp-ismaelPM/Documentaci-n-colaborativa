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
| 10   | Admin     | 192.168.10.0/24 |
| 20   | TI        | 192.168.20.0/24 |
| 30   | Invitados | 192.168.30.0/24 |