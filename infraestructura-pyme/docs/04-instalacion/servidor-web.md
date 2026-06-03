# Servidor Web LAMP — Documentación Técnica

> **Entorno:** Ubuntu Server 22.04 LTS · Apache 2.4 · PHP 8.1 · MySQL 8.0 / MariaDB 10.6  
> **Última revisión:** 2024

---

## Índice

1. [Requisitos previos](#1-requisitos-previos)
2. [Instalación del sistema base](#2-instalación-del-sistema-base)
3. [Apache — Servidor Web](#3-apache--servidor-web)
4. [PHP](#4-php)
5. [MySQL / MariaDB](#5-mysql--mariadb)
6. [Bases de datos de la PYME](#6-bases-de-datos-de-la-pyme)
7. [Acceso remoto seguro SSH](#7-acceso-remoto-seguro-ssh)
8. [Firewall UFW](#8-firewall-ufw)
9. [Monitorización con Netdata](#9-monitorización-con-netdata)
10. [Copias de seguridad automáticas](#10-copias-de-seguridad-automáticas)
11. [Estructura de directorios](#11-estructura-de-directorios)
12. [Referencias rápidas](#12-referencias-rápidas)

---

## 1. Requisitos previos

| Recurso | Mínimo recomendado |
|---|---|
| CPU | 2 vCPUs |
| RAM | 4 GB |
| Disco | 40 GB SSD |
| Red | IP estática o DDNS configurado |
| SO | Ubuntu Server 22.04 LTS (64 bit) |

Antes de comenzar, actualiza el sistema:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl wget gnupg2 software-properties-common unzip git
```

Configura el hostname:

```bash
sudo hostnamectl set-hostname servidor-pyme
echo "127.0.0.1 servidor-pyme" | sudo tee -a /etc/hosts
```

---

## 2. Instalación del sistema base

### Zona horaria

```bash
sudo timedatectl set-timezone Europe/Madrid
timedatectl status
```

### Sincronización NTP

```bash
sudo apt install -y systemd-timesyncd
sudo systemctl enable --now systemd-timesyncd
```

### Usuario de administración (no usar root directamente)

```bash
sudo adduser adminpyme
sudo usermod -aG sudo adminpyme
```

---

## 3. Apache — Servidor Web

### Instalación

```bash
sudo apt install -y apache2
sudo systemctl enable --now apache2
```

### Módulos esenciales

```bash
sudo a2enmod rewrite ssl headers deflate expires
sudo systemctl restart apache2
```

### VirtualHost para el sitio web de la PYME

Crea el fichero de configuración:

```bash
sudo nano /etc/apache2/sites-available/pyme.conf
```

Contenido:

```apache
<VirtualHost *:80>
    ServerName www.mipyme.local
    ServerAlias mipyme.local
    DocumentRoot /var/www/pyme/public

    <Directory /var/www/pyme/public>
        AllowOverride All
        Require all granted
        Options -Indexes +FollowSymLinks
    </Directory>

    ErrorLog  ${APACHE_LOG_DIR}/pyme_error.log
    CustomLog ${APACHE_LOG_DIR}/pyme_access.log combined

    # Redirigir a HTTPS cuando el certificado esté disponible
    # RewriteEngine On
    # RewriteRule ^(.*)$ https://%{HTTP_HOST}$1 [R=301,L]
</VirtualHost>
```

Activa el sitio y deshabilita el default:

```bash
sudo mkdir -p /var/www/pyme/public
sudo chown -R www-data:www-data /var/www/pyme
sudo a2ensite pyme.conf
sudo a2dissite 000-default.conf
sudo apache2ctl configtest
sudo systemctl reload apache2
```

### VirtualHost HTTPS (SSL)

```bash
sudo apt install -y certbot python3-certbot-apache
# Con dominio real:
sudo certbot --apache -d mipyme.com -d www.mipyme.com
# Renovación automática (ya incluida por certbot):
sudo systemctl status certbot.timer
```

Para entorno interno sin dominio público, genera un certificado autofirmado:

```bash
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/ssl/private/pyme-selfsigned.key \
  -out /etc/ssl/certs/pyme-selfsigned.crt \
  -subj "/C=ES/ST=Madrid/L=Madrid/O=MiPyme/CN=mipyme.local"
```

### Hardening básico de Apache

Edita `/etc/apache2/conf-available/security.conf`:

```apache
ServerTokens Prod
ServerSignature Off
TraceEnable Off
Header always set X-Content-Type-Options "nosniff"
Header always set X-Frame-Options "SAMEORIGIN"
Header always set X-XSS-Protection "1; mode=block"
Header always set Referrer-Policy "strict-origin-when-cross-origin"
```

```bash
sudo a2enconf security
sudo systemctl reload apache2
```

---

## 4. PHP

### Instalación de PHP 8.1 y extensiones

```bash
sudo apt install -y php8.1 php8.1-cli php8.1-fpm php8.1-mysql \
  php8.1-curl php8.1-gd php8.1-mbstring php8.1-xml php8.1-zip \
  php8.1-intl php8.1-bcmath php8.1-opcache libapache2-mod-php8.1
```

### Configuración de php.ini

Edita `/etc/php/8.1/apache2/php.ini` y ajusta:

```ini
; Límites de recursos
memory_limit = 256M
max_execution_time = 60
upload_max_filesize = 64M
post_max_size = 64M

; Seguridad
expose_php = Off
display_errors = Off
log_errors = On
error_log = /var/log/php/php_errors.log

; Zona horaria
date.timezone = Europe/Madrid

; OPcache
opcache.enable = 1
opcache.memory_consumption = 128
opcache.max_accelerated_files = 10000
opcache.revalidate_freq = 60
```

```bash
sudo mkdir -p /var/log/php
sudo chown www-data:www-data /var/log/php
sudo systemctl restart apache2
```

### Verificación

```bash
php -v
php -m | grep -E "mysql|curl|gd|mbstring|xml|zip"
```

Crea `/var/www/pyme/public/info.php` (solo para prueba, **eliminar en producción**):

```php
<?php phpinfo(); ?>
```

---

## 5. MySQL / MariaDB

### Instalación de MariaDB 10.6

```bash
sudo apt install -y mariadb-server mariadb-client
sudo systemctl enable --now mariadb
```

### Securización inicial

```bash
sudo mysql_secure_installation
```

Respuestas recomendadas:

| Pregunta | Respuesta |
|---|---|
| Set root password | Sí — contraseña robusta |
| Remove anonymous users | Sí |
| Disallow root login remotely | Sí |
| Remove test database | Sí |
| Reload privilege tables | Sí |

### Configuración de rendimiento

Edita `/etc/mysql/mariadb.conf.d/50-server.cnf`:

```ini
[mysqld]
# Motor por defecto
default_storage_engine = InnoDB

# Caché InnoDB (50-70% de la RAM disponible)
innodb_buffer_pool_size = 1G

# Logs
slow_query_log = 1
slow_query_log_file = /var/log/mysql/slow.log
long_query_time = 2

# Seguridad
bind-address = 127.0.0.1

# Juego de caracteres
character-set-server = utf8mb4
collation-server = utf8mb4_unicode_ci
```

```bash
sudo systemctl restart mariadb
```

---

## 6. Bases de datos de la PYME

### Base de datos — Sitio Web

```sql
-- Conectar como root
sudo mysql -u root -p

CREATE DATABASE db_web CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'user_web'@'localhost' IDENTIFIED BY 'CAMBIAR_PASSWORD_WEB';
GRANT ALL PRIVILEGES ON db_web.* TO 'user_web'@'localhost';
FLUSH PRIVILEGES;
```

### Base de datos — Gestión Interna

```sql
CREATE DATABASE db_gestion CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'user_gestion'@'localhost' IDENTIFIED BY 'CAMBIAR_PASSWORD_GESTION';
GRANT ALL PRIVILEGES ON db_gestion.* TO 'user_gestion'@'localhost';
FLUSH PRIVILEGES;

EXIT;
```

### Verificación

```bash
mysql -u user_web -p -e "SHOW DATABASES;"
mysql -u user_gestion -p -e "SHOW DATABASES;"
```

> **Importante:** Guarda las contraseñas en un gestor de contraseñas (ej. Bitwarden, KeePass). Nunca las escribas en el código fuente.

---

## 7. Acceso remoto seguro SSH

### Configuración del servidor SSH

Edita `/etc/ssh/sshd_config`:

```ini
# Puerto no estándar (oscuridad básica)
Port 2222

# Solo IPv4
AddressFamily inet

# Deshabilitar acceso root por SSH
PermitRootLogin no

# Solo autenticación por clave pública
PasswordAuthentication no
PubkeyAuthentication yes

# Usuarios permitidos
AllowUsers adminpyme

# Timeouts
ClientAliveInterval 300
ClientAliveCountMax 2
LoginGraceTime 30

# Deshabilitar características no usadas
X11Forwarding no
AllowAgentForwarding no
AllowTcpForwarding no
PermitTunnel no

# Banner de advertencia
Banner /etc/ssh/banner.txt
```

Crea el banner:

```bash
sudo bash -c 'cat > /etc/ssh/banner.txt << EOF
*************************************************************
*  Acceso restringido — Solo personal autorizado            *
*  Todos los accesos quedan registrados                     *
*************************************************************
EOF'
```

### Generar y copiar clave SSH (desde el equipo cliente)

```bash
# En el equipo del administrador:
ssh-keygen -t ed25519 -C "adminpyme@mipyme" -f ~/.ssh/id_pyme

# Copiar la clave pública al servidor:
ssh-copy-id -i ~/.ssh/id_pyme.pub -p 22 adminpyme@IP_SERVIDOR

# Tras verificar que funciona con clave, reiniciar SSH:
sudo systemctl restart sshd
```

### Fail2ban — Protección contra ataques de fuerza bruta

```bash
sudo apt install -y fail2ban
```

Crea `/etc/fail2ban/jail.local`:

```ini
[DEFAULT]
bantime  = 1h
findtime = 10m
maxretry = 5
destemail = admin@mipyme.com
action = %(action_mwl)s

[sshd]
enabled = true
port    = 2222
filter  = sshd
logpath = /var/log/auth.log
maxretry = 3
bantime  = 24h
```

```bash
sudo systemctl enable --now fail2ban
sudo fail2ban-client status sshd
```

---

## 8. Firewall UFW

### Configuración básica

```bash
# Política por defecto: denegar entrada, permitir salida
sudo ufw default deny incoming
sudo ufw default allow outgoing

# SSH (puerto personalizado)
sudo ufw allow 2222/tcp comment 'SSH'

# Web
sudo ufw allow 80/tcp comment 'HTTP'
sudo ufw allow 443/tcp comment 'HTTPS'

# Activar
sudo ufw enable
sudo ufw status verbose
```

### Reglas adicionales opcionales

```bash
# Permitir acceso a MySQL solo desde IP interna específica
sudo ufw allow from 192.168.1.0/24 to any port 3306 comment 'MySQL LAN'

# Limitar intentos SSH (rate limiting)
sudo ufw limit 2222/tcp
```

### Verificación del estado

```bash
sudo ufw status numbered
sudo ufw show listening
```

---

## 9. Monitorización con Netdata

### Instalación

```bash
curl https://get.netdata.cloud/kickstart.sh > /tmp/netdata-kickstart.sh
sh /tmp/netdata-kickstart.sh --non-interactive
```

### Acceso

Netdata escucha por defecto en el puerto **19999**. Accede desde la red local:

```
http://IP_SERVIDOR:19999
```

Para no exponer este puerto públicamente, abre solo desde la LAN:

```bash
sudo ufw allow from 192.168.1.0/24 to any port 19999 comment 'Netdata LAN'
```

O configura un túnel SSH para acceso externo seguro:

```bash
# Desde el equipo del administrador:
ssh -L 19999:localhost:19999 -p 2222 adminpyme@IP_SERVIDOR
# Luego abre http://localhost:19999 en tu navegador
```

### Alertas por correo

Edita `/etc/netdata/health_alarm_notify.conf`:

```bash
SEND_EMAIL="YES"
DEFAULT_RECIPIENT_EMAIL="admin@mipyme.com"
EMAIL_SENDER="netdata@mipyme.local"
```

### Script de chequeo rápido alternativo

Si no se desea Netdata, este script básico envía un resumen diario:

```bash
sudo nano /usr/local/bin/check-sistema.sh
```

```bash
#!/bin/bash
# check-sistema.sh — Chequeo básico del servidor PYME

FECHA=$(date '+%Y-%m-%d %H:%M')
EMAIL="admin@mipyme.com"
HOSTNAME=$(hostname)

# Uso de disco
DISCO=$(df -h / | awk 'NR==2{print $5}' | tr -d '%')
# Uso de RAM
RAM=$(free | awk '/Mem/{printf "%.0f", $3/$2*100}')
# Carga del sistema
CARGA=$(uptime | awk -F'load average:' '{print $2}' | xargs)
# Estado de servicios
APACHE=$(systemctl is-active apache2)
MYSQL=$(systemctl is-active mariadb)
SSH=$(systemctl is-active ssh)

CUERPO="=== Informe diario: $HOSTNAME ===
Fecha: $FECHA

RECURSOS:
  Disco /: ${DISCO}%
  RAM:     ${RAM}%
  Carga:   $CARGA

SERVICIOS:
  Apache:  $APACHE
  MariaDB: $MYSQL
  SSH:     $SSH

ALERTAS:"

[ "$DISCO" -gt 85 ] && CUERPO+="  ⚠️  Disco al ${DISCO}% — revisar espacio"$'\n'
[ "$RAM" -gt 90 ]   && CUERPO+="  ⚠️  RAM al ${RAM}% — revisar procesos"$'\n'
[ "$APACHE" != "active" ] && CUERPO+="  🔴 Apache CAÍDO"$'\n'
[ "$MYSQL"  != "active" ] && CUERPO+="  🔴 MariaDB CAÍDO"$'\n'

echo "$CUERPO" | mail -s "[$HOSTNAME] Informe diario $FECHA" "$EMAIL"
```

```bash
sudo chmod +x /usr/local/bin/check-sistema.sh
# Ejecutar cada día a las 07:00
echo "0 7 * * * root /usr/local/bin/check-sistema.sh" | sudo tee /etc/cron.d/check-sistema
```

---

## 10. Copias de seguridad automáticas

### Estructura de directorios de backup

```bash
sudo mkdir -p /backups/{bd/{diario,semanal,mensual},archivos/{diario,semanal}}
sudo chown -R root:root /backups
sudo chmod -R 700 /backups
```

### Script de backup de bases de datos

```bash
sudo nano /usr/local/bin/backup-bd.sh
```

```bash
#!/bin/bash
# backup-bd.sh — Volcado automático de bases de datos con rotación

set -euo pipefail

# ─── Configuración ────────────────────────────────────────────────
DB_USER="backup_user"
DB_PASS="CAMBIAR_PASSWORD_BACKUP"
BACKUP_DIR="/backups/bd"
FECHA=$(date +%Y%m%d_%H%M%S)
DIA_SEMANA=$(date +%u)   # 1=lunes … 7=domingo
DIA_MES=$(date +%d)
LOG="/var/log/backups.log"
BASES="db_web db_gestion"

# Rotación (días a conservar)
RETENER_DIARIO=7
RETENER_SEMANAL=4   # semanas
RETENER_MENSUAL=3   # meses
# ──────────────────────────────────────────────────────────────────

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" | tee -a "$LOG"; }

for BD in $BASES; do
    ARCHIVO="${BD}_${FECHA}.sql.gz"

    # Backup diario
    log "Volcando $BD..."
    mysqldump -u"$DB_USER" -p"$DB_PASS" \
        --single-transaction --quick --routines --triggers \
        "$BD" | gzip > "${BACKUP_DIR}/diario/${ARCHIVO}"
    log "OK: ${BACKUP_DIR}/diario/${ARCHIVO}"

    # Backup semanal (cada lunes)
    if [ "$DIA_SEMANA" -eq 1 ]; then
        cp "${BACKUP_DIR}/diario/${ARCHIVO}" "${BACKUP_DIR}/semanal/"
        log "Copia semanal guardada"
    fi

    # Backup mensual (día 1 del mes)
    if [ "$DIA_MES" -eq "01" ]; then
        cp "${BACKUP_DIR}/diario/${ARCHIVO}" "${BACKUP_DIR}/mensual/"
        log "Copia mensual guardada"
    fi
done

# Rotación: eliminar backups antiguos
find "${BACKUP_DIR}/diario"   -name "*.sql.gz" -mtime +${RETENER_DIARIO}   -delete
find "${BACKUP_DIR}/semanal"  -name "*.sql.gz" -mtime +$((RETENER_SEMANAL * 7))  -delete
find "${BACKUP_DIR}/mensual"  -name "*.sql.gz" -mtime +$((RETENER_MENSUAL * 30)) -delete

log "Backup de bases de datos completado."
```

### Script de backup de archivos web

```bash
sudo nano /usr/local/bin/backup-archivos.sh
```

```bash
#!/bin/bash
# backup-archivos.sh — Sincronización de archivos web con rsync

set -euo pipefail

ORIGEN="/var/www/pyme"
DESTINO_LOCAL="/backups/archivos"
FECHA=$(date +%Y%m%d)
LOG="/var/log/backups.log"

# Destino remoto (descomentar y configurar si se dispone de servidor externo)
# REMOTO="backup@servidor-remoto:/backups/pyme"

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" | tee -a "$LOG"; }

log "Iniciando backup de archivos web..."

# Backup incremental local
rsync -avz --delete \
    --exclude="*.log" \
    --exclude=".git" \
    --exclude="node_modules" \
    --link-dest="${DESTINO_LOCAL}/diario/ultimo" \
    "$ORIGEN/" \
    "${DESTINO_LOCAL}/diario/${FECHA}/"

# Actualizar enlace al último backup
ln -sfn "${DESTINO_LOCAL}/diario/${FECHA}" "${DESTINO_LOCAL}/diario/ultimo"

# Sincronización remota (descomentar si hay servidor externo)
# rsync -avz -e "ssh -p 2222 -i /root/.ssh/id_backup" \
#     "${DESTINO_LOCAL}/diario/${FECHA}/" \
#     "$REMOTO/"

# Eliminar backups de archivos con más de 14 días
find "${DESTINO_LOCAL}/diario" -maxdepth 1 -type d -name "????????" \
    -mtime +14 -exec rm -rf {} +

log "Backup de archivos completado."
```

### Usuario específico para backups de BD

```sql
sudo mysql -u root -p

CREATE USER 'backup_user'@'localhost' IDENTIFIED BY 'CAMBIAR_PASSWORD_BACKUP';
GRANT SELECT, RELOAD, SHOW DATABASES, LOCK TABLES, REPLICATION CLIENT ON *.* TO 'backup_user'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

### Permisos y programación con cron

```bash
sudo chmod +x /usr/local/bin/backup-bd.sh
sudo chmod +x /usr/local/bin/backup-archivos.sh
touch /var/log/backups.log
sudo chmod 640 /var/log/backups.log

# Programar en cron
sudo crontab -e
```

Añadir:

```cron
# Backup de BD: cada día a las 02:00
0 2 * * * /usr/local/bin/backup-bd.sh

# Backup de archivos: cada día a las 03:00
0 3 * * * /usr/local/bin/backup-archivos.sh
```

### Verificación manual

```bash
sudo /usr/local/bin/backup-bd.sh
sudo /usr/local/bin/backup-archivos.sh
ls -lh /backups/bd/diario/
cat /var/log/backups.log
```

---

## 11. Estructura de directorios

```
/
├── var/
│   ├── www/
│   │   └── pyme/
│   │       └── public/          ← DocumentRoot del sitio web
│   └── log/
│       ├── apache2/             ← Logs de Apache
│       ├── mysql/               ← Logs de MariaDB
│       └── php/                 ← Logs de PHP
├── etc/
│   ├── apache2/
│   │   └── sites-available/pyme.conf
│   ├── mysql/
│   │   └── mariadb.conf.d/
│   ├── ssh/
│   │   └── sshd_config
│   └── ufw/
├── backups/
│   ├── bd/
│   │   ├── diario/
│   │   ├── semanal/
│   │   └── mensual/
│   └── archivos/
│       └── diario/
└── usr/local/bin/
    ├── backup-bd.sh
    ├── backup-archivos.sh
    └── check-sistema.sh
```

---

## 12. Referencias rápidas

### Comandos de gestión de servicios

```bash
# Estado general
sudo systemctl status apache2 mariadb ssh ufw fail2ban

# Reiniciar servicios
sudo systemctl restart apache2
sudo systemctl restart mariadb

# Ver logs en tiempo real
sudo tail -f /var/log/apache2/pyme_error.log
sudo tail -f /var/log/mysql/error.log
sudo journalctl -u ssh -f

# Comprobar configuración Apache
sudo apache2ctl configtest

# Comprobar configuración SSH
sudo sshd -t
```

### Puertos en uso

| Puerto | Servicio | Acceso |
|---|---|---|
| 2222 | SSH | Administradores |
| 80 | HTTP | Público |
| 443 | HTTPS | Público |
| 3306 | MariaDB | Solo localhost |
| 19999 | Netdata | Solo LAN |

### Usuarios del sistema

| Usuario | Rol |
|---|---|
| `adminpyme` | Administración SSH |
| `www-data` | Proceso Apache/PHP |
| `user_web` | Acceso BD web |
| `user_gestion` | Acceso BD gestión |
| `backup_user` | Solo lectura para backups |
