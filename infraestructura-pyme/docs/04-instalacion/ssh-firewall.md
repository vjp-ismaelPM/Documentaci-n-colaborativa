# Acceso Remoto SSH y Firewall UFW

**Proyecto:** Infraestructura LAMP – PYME  
**Versión:** 1.0  
**Última actualización:** 2025-06  
**Responsable:** Documentalista de Plataforma / Operaciones

---

## Índice

1. [Descripción general](#1-descripción-general)
2. [Configuración de SSH](#2-configuración-de-ssh)
3. [Autenticación con clave pública](#3-autenticación-con-clave-pública)
4. [Hardening del servidor SSH](#4-hardening-del-servidor-ssh)
5. [Configuración del firewall con UFW](#5-configuración-del-firewall-con-ufw)
6. [Reglas UFW detalladas](#6-reglas-ufw-detalladas)
7. [Verificación del firewall](#7-verificación-del-firewall)
8. [Fail2ban: protección contra fuerza bruta](#8-fail2ban-protección-contra-fuerza-bruta)
9. [Procedimiento de acceso para administradores](#9-procedimiento-de-acceso-para-administradores)
10. [Buenas prácticas de seguridad](#10-buenas-prácticas-de-seguridad)
11. [Resolución de problemas comunes](#11-resolución-de-problemas-comunes)

---

## 1. Descripción general

Este documento describe la configuración del acceso remoto seguro mediante **SSH** y la protección del servidor con el firewall **UFW (Uncomplicated Firewall)** en Ubuntu Server 22.04 LTS.

### Política de seguridad aplicada

| Acceso | Política |
|---|---|
| SSH (puerto 22) | Solo desde la red de la oficina (`192.168.1.0/24`) |
| HTTP (puerto 80) | Abierto a todo el tráfico (a través de HAProxy) |
| HTTPS (puerto 443) | Abierto a todo el tráfico (a través de HAProxy) |
| MySQL (puerto 3306) | Solo localhost, nunca desde exterior |
| Resto de puertos | Denegado por defecto |

```
[Administrador] ──SSH──► [Puerto 22] (solo desde 192.168.1.0/24)
[Internet]      ──HTTP──► [Puerto 80/443] → HAProxy → Apache
[Exterior]      ──XXXX──► [Puerto 3306] ✗ DENEGADO
```

---

## 2. Configuración de SSH

### 2.1 Verificar que SSH está instalado y activo

```bash
sudo apt install openssh-server -y
sudo systemctl status ssh
sudo systemctl enable ssh
```

### 2.2 Verificar el puerto de escucha

```bash
sudo ss -tlnp | grep ':22'
# LISTEN  0  128  0.0.0.0:22  ...
```

### 2.3 Conectarse al servidor por primera vez

Desde el equipo del administrador:

```bash
ssh usuario@192.168.1.X
```

---

## 3. Autenticación con clave pública

El acceso con contraseña será deshabilitado. Solo se permitirá acceso mediante **par de claves SSH**.

### 3.1 Generar el par de claves en el equipo del administrador

```bash
# En el equipo local del administrador (no en el servidor)
ssh-keygen -t ed25519 -C "admin@empresa-pyme" -f ~/.ssh/id_empresa
```

Esto genera dos archivos:
- `~/.ssh/id_empresa` → clave privada (nunca compartir)
- `~/.ssh/id_empresa.pub` → clave pública (se copia al servidor)

### 3.2 Copiar la clave pública al servidor

```bash
ssh-copy-id -i ~/.ssh/id_empresa.pub usuario@192.168.1.X
```

O manualmente:

```bash
# En el servidor
mkdir -p ~/.ssh
chmod 700 ~/.ssh
nano ~/.ssh/authorized_keys
# Pegar el contenido del archivo .pub
chmod 600 ~/.ssh/authorized_keys
```

### 3.3 Verificar el acceso con clave

```bash
ssh -i ~/.ssh/id_empresa usuario@192.168.1.X
# Debe conectar sin pedir contraseña
```

---

## 4. Hardening del servidor SSH

Editar la configuración de SSH:

```bash
sudo nano /etc/ssh/sshd_config
```

Aplicar los siguientes cambios:

```ini
# Puerto estándar (considerar cambiarlo en entornos más expuestos)
Port 22

# Solo IPv4 (simplifica las reglas de firewall)
AddressFamily inet

# Deshabilitar login de root por SSH
PermitRootLogin no

# Solo autenticación con clave pública
PasswordAuthentication no
PubkeyAuthentication yes
AuthorizedKeysFile .ssh/authorized_keys

# Deshabilitar autenticación vacía y por desafío
PermitEmptyPasswords no
ChallengeResponseAuthentication no

# Limitar reintentos de autenticación
MaxAuthTries 3

# Tiempo de gracia para autenticarse (segundos)
LoginGraceTime 30

# Desconectar sesiones inactivas tras 10 minutos
ClientAliveInterval 600
ClientAliveCountMax 0

# Deshabilitar reenvío de X11 (innecesario en servidor)
X11Forwarding no

# Usuarios permitidos (añadir aquí los administradores)
AllowUsers adminpyme
```

Aplicar los cambios:

```bash
sudo sshd -t          # Verificar sintaxis antes de reiniciar
sudo systemctl restart ssh
```

> ⚠️ **No cerrar la sesión actual** hasta verificar que se puede abrir una nueva sesión correctamente.

---

## 5. Configuración del firewall con UFW

### 5.1 Instalar UFW (suele venir preinstalado)

```bash
sudo apt install ufw -y
```

### 5.2 Política por defecto: denegar todo el tráfico entrante

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
```

---

## 6. Reglas UFW detalladas

### 6.1 Permitir SSH solo desde la red de la oficina

```bash
# Acceso SSH restringido a la red interna de la empresa
sudo ufw allow from 192.168.1.0/24 to any port 22 comment 'SSH administracion oficina'
```

> ⚠️ **Crítico:** añadir esta regla **antes** de activar UFW para no perder el acceso al servidor.

### 6.2 Permitir tráfico web (HTTP y HTTPS)

```bash
# Tráfico web público a través de HAProxy
sudo ufw allow 80/tcp comment 'HTTP publico'
sudo ufw allow 443/tcp comment 'HTTPS publico'
```

### 6.3 Activar el firewall

```bash
sudo ufw enable
# Command may disrupt existing ssh connections. Proceed with operation (y|n)? y
```

### 6.4 Reglas opcionales adicionales

```bash
# Permitir MySQL solo desde la red interna (si se necesita acceso remoto a BD)
# sudo ufw allow from 192.168.1.0/24 to any port 3306 comment 'MySQL red interna'

# Permitir acceso a Netdata solo desde la red interna
# sudo ufw allow from 192.168.1.0/24 to any port 19999 comment 'Netdata monitorización'
```

---

## 7. Verificación del firewall

### 7.1 Ver el estado y las reglas activas

```bash
sudo ufw status verbose
```

Salida esperada:

```
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), disabled (routed)

To                         Action      From
--                         ------      ----
22/tcp                     ALLOW IN    192.168.1.0/24    # SSH administracion oficina
80/tcp                     ALLOW IN    Anywhere          # HTTP publico
443/tcp                    ALLOW IN    Anywhere          # HTTPS publico
```

### 7.2 Ver reglas numeradas (para eliminar una específica)

```bash
sudo ufw status numbered
```

### 7.3 Eliminar una regla por número

```bash
sudo ufw delete 3   # elimina la regla número 3
```

### 7.4 Verificar conectividad desde exterior

```bash
# Desde un equipo fuera de la red 192.168.1.0/24
ssh usuario@IP_publica_servidor
# Debe ser rechazado: Connection refused o timeout
```

---

## 8. Fail2ban: protección contra fuerza bruta

Fail2ban monitoriza los logs de SSH y banea automáticamente IPs que realizan demasiados intentos fallidos de conexión.

### 8.1 Instalación

```bash
sudo apt install fail2ban -y
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
```

### 8.2 Configuración personalizada

Crear un archivo de configuración local (no editar el original para preservarlo en actualizaciones):

```bash
sudo nano /etc/fail2ban/jail.local
```

```ini
[DEFAULT]
bantime  = 3600        # Baneo de 1 hora
findtime = 600         # Ventana de observación: 10 minutos
maxretry = 5           # Máximo de intentos fallidos

[sshd]
enabled  = true
port     = 22
filter   = sshd
logpath  = /var/log/auth.log
maxretry = 3
bantime  = 7200        # Baneo de 2 horas para SSH
```

### 8.3 Aplicar y verificar

```bash
sudo systemctl restart fail2ban
sudo fail2ban-client status
sudo fail2ban-client status sshd
```

### 8.4 Desbanear una IP manualmente (si hay falso positivo)

```bash
sudo fail2ban-client set sshd unbanip 192.168.1.X
```

---

## 9. Procedimiento de acceso para administradores

### Acceso habitual desde la oficina

```bash
ssh -i ~/.ssh/id_empresa adminpyme@192.168.1.X
```

### Acceso desde fuera de la oficina (si es necesario)

Opciones recomendadas por orden de preferencia:

1. **VPN** conectando primero a la red de la oficina (recomendado).
2. **Túnel SSH** a través de un servidor intermedio (jump host).
3. Añadir temporalmente la IP externa a UFW y retirarla al finalizar:

```bash
# Añadir IP temporal
sudo ufw allow from 85.100.X.X to any port 22 comment 'Acceso temporal admin remoto'

# Eliminar tras finalizar el trabajo
sudo ufw status numbered
sudo ufw delete [número_de_la_regla]
```

---

## 10. Buenas prácticas de seguridad

- **Nunca usar contraseñas para SSH:** solo claves pública/privada.
- **Proteger la clave privada con passphrase** al generarla con `ssh-keygen`.
- **No conectarse como root** por SSH bajo ninguna circunstancia.
- **Revisar periódicamente los logs** de acceso:

```bash
sudo grep 'Failed password\|Accepted publickey' /var/log/auth.log | tail -20
```

- **Rotar las claves SSH** al menos una vez al año o cuando un administrador deje la empresa.
- **Mantener UFW y Fail2ban actualizados:**

```bash
sudo apt update && sudo apt upgrade ufw fail2ban -y
```

- **Documentar cada regla UFW** con el parámetro `comment` para facilitar auditorías futuras.

---

## 11. Resolución de problemas comunes

| Síntoma | Causa probable | Solución |
|---|---|---|
| `Connection refused` en SSH | UFW bloqueando o SSH no activo | `sudo systemctl status ssh` y revisar reglas UFW |
| `Permission denied (publickey)` | Clave no autorizada o permisos incorrectos | Verificar `~/.ssh/authorized_keys` y permisos `600` |
| IP baneada por Fail2ban | Demasiados intentos fallidos | `sudo fail2ban-client set sshd unbanip <IP>` |
| UFW bloquea tráfico legítimo | Regla demasiado restrictiva | `sudo ufw status numbered` y revisar/ajustar reglas |
| No se puede activar UFW | Puede dejar sin acceso SSH | Añadir regla SSH **antes** de `ufw enable` |

---

## Referencias

- [Documentación UFW – Ubuntu](https://help.ubuntu.com/community/UFW)
- [OpenSSH Manual](https://www.openssh.com/manual.html)
- [Fail2ban Documentation](https://www.fail2ban.org/wiki/index.php/MANUAL_0_8)
- → Ver también: [`servidor-web.md`](servidor-web.md) | [`base-de-datos.md`](base-de-datos.md) | [`monitorizacion.md`](monitorizacion.md)

## Reglas UFW
- Permitir SSH solo desde IP de la oficina: `ufw allow from 192.168.1.0/24 to any port 22`
- Permitir tráfico web: `ufw allow 80/tcp` y `ufw allow 443/tcp`
