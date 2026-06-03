# 05 — Operación, Mantenimiento y Recuperación ante Desastres

> **Documento destinado al cliente**  
> Guía de mantenimiento diario, procedimientos de recuperación y plan de continuidad del negocio para el servidor LAMP de la PYME.

---

## Índice

1. [Acceso al servidor](#1-acceso-al-servidor)
2. [Tareas de mantenimiento rutinario](#2-tareas-de-mantenimiento-rutinario)
3. [Gestión de servicios](#3-gestión-de-servicios)
4. [Monitorización y alertas](#4-monitorización-y-alertas)
5. [Gestión de copias de seguridad](#5-gestión-de-copias-de-seguridad)
6. [Actualizaciones del sistema](#6-actualizaciones-del-sistema)
7. [Resolución de problemas frecuentes](#7-resolución-de-problemas-frecuentes)
8. [Plan de recuperación ante desastres (DRP)](#8-plan-de-recuperación-ante-desastres-drp)
9. [Contactos y escalado](#9-contactos-y-escalado)
10. [Registro de cambios](#10-registro-de-cambios)

---

## 1. Acceso al servidor

### Conexión SSH

```bash
# Desde Linux / macOS
ssh -p 2222 -i ~/.ssh/id_pyme adminpyme@IP_SERVIDOR

# Desde Windows (PowerShell o PuTTY)
ssh -p 2222 -i C:\Users\TuUsuario\.ssh\id_pyme adminpyme@IP_SERVIDOR
```

> Si no tienes la clave SSH configurada, solicítala al administrador técnico. **No se permite acceso por contraseña.**

### Elevar privilegios

```bash
# Ejecutar un comando puntual como root
sudo comando

# Abrir sesión root (solo si es imprescindible)
sudo -i
```

---

## 2. Tareas de mantenimiento rutinario

### Calendario de tareas

| Frecuencia | Tarea | Responsable |
|---|---|---|
| Diaria | Revisar log de backups | Admin |
| Diaria | Revisar alertas de Netdata | Admin |
| Semanal | Revisar logs de errores Apache y MySQL | Admin |
| Semanal | Verificar espacio en disco | Admin |
| Mensual | Aplicar actualizaciones de seguridad | Admin |
| Mensual | Restaurar un backup de prueba | Admin |
| Trimestral | Revisar usuarios y permisos | Admin |
| Anual | Renovar certificados SSL (si no es automático) | Admin |

### Chequeo diario rápido (5 minutos)

```bash
# 1. Estado de los servicios críticos
sudo systemctl status apache2 mariadb ssh --no-pager

# 2. Espacio en disco
df -h /

# 3. Uso de RAM
free -h

# 4. Carga del sistema
uptime

# 5. Últimas líneas del log de backups
tail -20 /var/log/backups.log

# 6. Intentos de acceso fallidos SSH
sudo grep "Failed password\|Invalid user" /var/log/auth.log | tail -20

# 7. IPs baneadas por Fail2ban
sudo fail2ban-client status sshd
```

### Chequeo semanal (15-20 minutos)

```bash
# Errores de Apache de la última semana
sudo grep "error" /var/log/apache2/pyme_error.log | \
  awk -v date="$(date -d '7 days ago' '+%a %b')" '$0 ~ date' | tail -50

# Queries lentas en MariaDB
sudo tail -100 /var/log/mysql/slow.log

# Listar backups disponibles
ls -lh /backups/bd/diario/
ls -lh /backups/bd/semanal/
ls -lh /backups/bd/mensual/

# Ver IPs más activas en el servidor web
sudo awk '{print $1}' /var/log/apache2/pyme_access.log | \
  sort | uniq -c | sort -rn | head -20
```

---

## 3. Gestión de servicios

### Apache

```bash
# Iniciar / parar / reiniciar / recargar configuración
sudo systemctl start apache2
sudo systemctl stop apache2
sudo systemctl restart apache2      # Interrupción breve del servicio
sudo systemctl reload apache2       # Sin interrupción (recarga config)

# Comprobar configuración antes de aplicar
sudo apache2ctl configtest

# Habilitar / deshabilitar sitio
sudo a2ensite nombre.conf
sudo a2dissite nombre.conf

# Habilitar / deshabilitar módulo
sudo a2enmod nombre_modulo
sudo a2dismod nombre_modulo

# Logs en tiempo real
sudo tail -f /var/log/apache2/pyme_error.log
sudo tail -f /var/log/apache2/pyme_access.log
```

### MariaDB

```bash
# Iniciar / parar / reiniciar
sudo systemctl start mariadb
sudo systemctl stop mariadb
sudo systemctl restart mariadb

# Conectar a la consola de base de datos
sudo mysql -u root -p
mysql -u user_web -p db_web
mysql -u user_gestion -p db_gestion

# Ver estado del motor
sudo mysqladmin -u root -p status
sudo mysqladmin -u root -p extended-status | grep -i "query\|thread\|conn"

# Ver logs de errores
sudo tail -f /var/log/mysql/error.log
```

### SSH y Firewall

```bash
# Estado del firewall
sudo ufw status verbose

# Añadir regla puntual
sudo ufw allow from IP_CONFIANZA to any port 2222

# Eliminar regla
sudo ufw status numbered
sudo ufw delete NUMERO_REGLA

# Estado de Fail2ban y desbanear IP
sudo fail2ban-client status sshd
sudo fail2ban-client set sshd unbanip IP_A_DESBANEAR
```

---

## 4. Monitorización y alertas

### Panel Netdata

Accede al panel web desde la red local:

```
http://IP_SERVIDOR:19999
```

O mediante túnel SSH desde fuera de la red:

```bash
ssh -L 19999:localhost:19999 -p 2222 adminpyme@IP_SERVIDOR
# Luego abre: http://localhost:19999
```

**Métricas clave a vigilar:**

| Métrica | Umbral de alerta | Acción |
|---|---|---|
| Uso CPU | > 80% sostenido | Revisar procesos con `top` o `htop` |
| Uso RAM | > 90% | Revisar `free -h`, reiniciar servicios |
| Uso disco | > 85% | Limpiar logs antiguos, ampliar disco |
| Tiempo respuesta web | > 2s | Revisar logs Apache y queries lentas |
| Conexiones BD activas | > 100 | Revisar aplicación, posible leak |

### Revisar alertas activas de Netdata

```bash
# Ver alertas activas desde terminal
sudo /usr/libexec/netdata/plugins.d/alarm-notify.sh test
curl -s http://localhost:19999/api/v1/alarms | python3 -m json.tool | grep -A3 '"status"'
```

---

## 5. Gestión de copias de seguridad

### Verificar que los backups se están realizando

```bash
# Ver log de backups
cat /var/log/backups.log

# Listar backups por tipo
echo "=== DIARIOS ===" && ls -lh /backups/bd/diario/
echo "=== SEMANALES ===" && ls -lh /backups/bd/semanal/
echo "=== MENSUALES ===" && ls -lh /backups/bd/mensual/
```

### Ejecutar backup manual

```bash
# Backup de bases de datos
sudo /usr/local/bin/backup-bd.sh

# Backup de archivos web
sudo /usr/local/bin/backup-archivos.sh
```

### Restaurar una base de datos

> ⚠️ **Atención:** La restauración sobreescribe los datos actuales. Haz un backup previo antes de restaurar.

```bash
# 1. Hacer backup de seguridad antes de restaurar
sudo mysqldump -u root -p --single-transaction db_web | gzip > \
  /backups/bd/pre-restauracion_$(date +%Y%m%d_%H%M%S).sql.gz

# 2. Identificar el archivo a restaurar
ls -lh /backups/bd/diario/
# Ejemplo: db_web_20240315_020001.sql.gz

# 3. Restaurar
gunzip -c /backups/bd/diario/db_web_20240315_020001.sql.gz | \
  sudo mysql -u root -p db_web

# 4. Verificar
sudo mysql -u root -p -e "USE db_web; SHOW TABLES;"
```

### Restaurar archivos web

```bash
# Ver backups disponibles
ls /backups/archivos/diario/

# Restaurar desde una fecha concreta (ej. 20240315)
sudo rsync -avz --delete \
  /backups/archivos/diario/20240315/ \
  /var/www/pyme/

# Restaurar un archivo concreto
sudo cp /backups/archivos/diario/20240315/public/index.php \
  /var/www/pyme/public/index.php

# Corregir permisos tras restauración
sudo chown -R www-data:www-data /var/www/pyme
sudo find /var/www/pyme -type d -exec chmod 755 {} \;
sudo find /var/www/pyme -type f -exec chmod 644 {} \;
```

---

## 6. Actualizaciones del sistema

### Actualización de seguridad mensual

```bash
# 1. Hacer backup completo antes de actualizar
sudo /usr/local/bin/backup-bd.sh
sudo /usr/local/bin/backup-archivos.sh

# 2. Actualizar lista de paquetes
sudo apt update

# 3. Ver qué se va a actualizar
apt list --upgradable

# 4. Aplicar actualizaciones
sudo apt upgrade -y

# 5. Actualizaciones de seguridad urgentes solamente
sudo apt install -y unattended-upgrades
sudo unattended-upgrades --dry-run  # Ver qué aplicaría

# 6. Reiniciar servicios afectados (o el servidor si es necesario)
sudo systemctl restart apache2 mariadb

# 7. Verificar que todo sigue funcionando
sudo systemctl status apache2 mariadb ssh
curl -I http://localhost
```

### Actualizaciones automáticas de seguridad

```bash
sudo dpkg-reconfigure -plow unattended-upgrades
# Seleccionar "Yes" para activar actualizaciones de seguridad automáticas
```

Edita `/etc/apt/apt.conf.d/50unattended-upgrades` para recibir notificaciones:

```
Unattended-Upgrade::Mail "admin@mipyme.com";
Unattended-Upgrade::MailOnlyOnError "false";
Unattended-Upgrade::Remove-Unused-Dependencies "true";
```

---

## 7. Resolución de problemas frecuentes

### El sitio web no carga (502 / 500 / pantalla en blanco)

```bash
# 1. Verificar estado de Apache
sudo systemctl status apache2

# 2. Ver errores recientes
sudo tail -50 /var/log/apache2/pyme_error.log

# 3. Verificar PHP
php -l /var/www/pyme/public/index.php   # Comprobar sintaxis

# 4. Verificar permisos
ls -la /var/www/pyme/public/
sudo chown -R www-data:www-data /var/www/pyme

# 5. Reiniciar Apache
sudo systemctl restart apache2
```

### La base de datos no responde

```bash
# 1. Estado del servicio
sudo systemctl status mariadb

# 2. Ver log de errores
sudo tail -50 /var/log/mysql/error.log

# 3. Intentar conexión
sudo mysql -u root -p -e "SELECT 1;"

# 4. Reparar tablas corruptas (MyISAM)
sudo mysqlcheck -u root -p --auto-repair --all-databases

# 5. Reiniciar MariaDB
sudo systemctl restart mariadb
```

### Disco lleno

```bash
# 1. Identificar qué ocupa más espacio
sudo du -sh /* | sort -rh | head -20
sudo du -sh /var/log/* | sort -rh | head -10

# 2. Limpiar logs antiguos de Apache
sudo find /var/log/apache2 -name "*.gz" -mtime +30 -delete
sudo truncate -s 0 /var/log/apache2/pyme_access.log

# 3. Limpiar caché de APT
sudo apt autoremove -y
sudo apt autoclean

# 4. Eliminar backups muy antiguos manualmente
ls /backups/bd/diario/
sudo rm /backups/bd/diario/ARCHIVO_MUY_ANTIGUO.sql.gz

# 5. Ver uso tras limpiar
df -h /
```

### SSH: no puedo conectarme

```bash
# Desde el servidor (acceso local o consola del VPS):

# 1. Verificar que SSH está activo
sudo systemctl status ssh

# 2. Ver si la IP está baneada
sudo fail2ban-client status sshd

# 3. Desbanear tu IP
sudo fail2ban-client set sshd unbanip TU_IP

# 4. Verificar reglas de firewall
sudo ufw status

# 5. Verificar que el puerto está escuchando
sudo ss -tlnp | grep 2222
```

### Certificado SSL caducado

```bash
# Ver fecha de caducidad
sudo openssl x509 -noout -dates -in /etc/letsencrypt/live/mipyme.com/cert.pem

# Renovar manualmente
sudo certbot renew --force-renewal

# Verificar renovación automática
sudo certbot renew --dry-run
sudo systemctl status certbot.timer
```

---

## 8. Plan de recuperación ante desastres (DRP)

### Niveles de incidencia

| Nivel | Descripción | Tiempo máximo de respuesta | Tiempo máximo de recuperación |
|---|---|---|---|
| 🔴 Crítico | Servidor caído, datos inaccesibles | 1 hora | 4 horas |
| 🟠 Alto | Servicio web caído, BD funcional | 2 horas | 8 horas |
| 🟡 Medio | Degradación del rendimiento | 4 horas | 24 horas |
| 🟢 Bajo | Problema menor, servicio funcional | 24 horas | 72 horas |

---

### Escenario 1 — Fallo de un servicio (Apache, MariaDB)

**Síntoma:** El sitio web no responde o da error de BD.

**Procedimiento:**

```bash
# Paso 1: Verificar el servicio afectado
sudo systemctl status apache2
sudo systemctl status mariadb

# Paso 2: Intentar reinicio
sudo systemctl restart apache2
sudo systemctl restart mariadb

# Paso 3: Revisar logs para encontrar la causa
sudo journalctl -u apache2 --since "1 hour ago"
sudo journalctl -u mariadb --since "1 hour ago"

# Paso 4: Si el servicio no arranca, revisar configuración
sudo apache2ctl configtest
sudo mysqld --user=mysql --verbose --help 2>&1 | head -20

# Paso 5: Si hay corrupción de BD, restaurar último backup
# → Ver sección 5: Restaurar una base de datos
```

**Estimación de recuperación:** 15-60 minutos.

---

### Escenario 2 — Servidor inaccesible (fallo hardware o SO)

**Síntoma:** No hay respuesta a ping ni SSH.

**Procedimiento:**

1. **Acceder a la consola del VPS** desde el panel de control del proveedor (no requiere red).
2. Verificar si el servidor está encendido y reiniciarlo si es necesario.
3. Comprobar el estado del sistema desde la consola:
   ```bash
   systemctl list-failed
   dmesg | tail -30
   journalctl -p err --since "2 hours ago"
   ```
4. Si el sistema operativo está corrupto, proceder al **Escenario 3**.

**Estimación de recuperación:** 1-3 horas.

---

### Escenario 3 — Pérdida total del servidor (reinstalación completa)

**Síntoma:** El servidor no arranca, disco corrupto, o pérdida del VPS.

**Procedimiento completo de reinstalación:**

#### Fase 1 — Preparar nuevo servidor (30 min)

```bash
# En el nuevo servidor Ubuntu Server 22.04:
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl wget git unzip

# Configurar usuario admin
sudo adduser adminpyme
sudo usermod -aG sudo adminpyme
```

#### Fase 2 — Reinstalar la pila LAMP (45 min)

Seguir el documento `servidor-web.md` desde el punto 3 al 6.

```bash
sudo apt install -y apache2 php8.1 php8.1-mysql mariadb-server
# Configurar VirtualHost, PHP, MariaDB según servidor-web.md
```

#### Fase 3 — Restaurar bases de datos (15 min)

```bash
# Transferir backup desde ubicación externa
scp -P 22 backup@servidor-remoto:/backups/bd/semanal/BACKUP.sql.gz /tmp/

# O desde copia local si se dispone de acceso al disco original
# Restaurar
gunzip -c /tmp/db_web_FECHA.sql.gz | sudo mysql -u root -p db_web
gunzip -c /tmp/db_gestion_FECHA.sql.gz | sudo mysql -u root -p db_gestion
```

#### Fase 4 — Restaurar archivos web (20 min)

```bash
# Transferir y descomprimir archivos
rsync -avz backup@servidor-remoto:/backups/archivos/diario/ultimo/ /var/www/pyme/
sudo chown -R www-data:www-data /var/www/pyme
```

#### Fase 5 — Restaurar configuración (20 min)

```bash
# SSH (copiar claves autorizadas)
mkdir -p /home/adminpyme/.ssh
# Pegar clave pública del administrador
nano /home/adminpyme/.ssh/authorized_keys
chmod 700 /home/adminpyme/.ssh
chmod 600 /home/adminpyme/.ssh/authorized_keys

# Firewall
sudo ufw default deny incoming
sudo ufw allow 2222/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable

# Fail2ban
sudo apt install -y fail2ban
# Restaurar /etc/fail2ban/jail.local

# Reinstalar Netdata
curl https://get.netdata.cloud/kickstart.sh | sh

# Restaurar cron de backups
sudo crontab -e
# Añadir las líneas de backup (ver servidor-web.md sección 10)
```

#### Fase 6 — Verificación final (15 min)

```bash
# Servicios
sudo systemctl status apache2 mariadb ssh ufw fail2ban

# Acceso web
curl -I http://localhost

# Acceso BD
mysql -u user_web -p -e "USE db_web; SHOW TABLES;"

# Backup manual de prueba
sudo /usr/local/bin/backup-bd.sh
```

**Estimación total de recuperación:** 2-4 horas.

---

### Escenario 4 — Ataque o intrusión

**Síntoma:** Comportamiento anómalo, ficheros modificados, tráfico inusual, mensajes de ransomware.

**Procedimiento de contención:**

```bash
# 1. AISLAR el servidor inmediatamente (desde consola del proveedor)
#    Desactivar interfaz de red o bloquear todo el tráfico:
sudo ufw default deny incoming
sudo ufw default deny outgoing
sudo ufw enable

# 2. Preservar evidencias ANTES de tocar nada
sudo cp /var/log/auth.log /tmp/evidencia_auth_$(date +%Y%m%d).log
sudo cp /var/log/apache2/pyme_access.log /tmp/evidencia_apache_$(date +%Y%m%d).log
sudo last -50 > /tmp/evidencia_last.txt
sudo ps aux > /tmp/evidencia_procesos.txt
sudo netstat -tulnp > /tmp/evidencia_puertos.txt

# 3. Identificar el vector de ataque
sudo grep -i "POST\|eval\|base64\|exec" /var/log/apache2/pyme_access.log | tail -50
sudo find /var/www -name "*.php" -newer /var/www/pyme/public/index.php -ls
sudo find /tmp /var/tmp -type f -ls

# 4. Notificar al proveedor y al responsable de la empresa
```

**Tras el análisis:**

1. Restaurar desde el último backup limpio (anterior al ataque).
2. Cambiar **todas** las contraseñas (BD, SSH, panel de control).
3. Revocar y regenerar claves SSH.
4. Identificar y corregir la vulnerabilidad explotada.
5. Aplicar todas las actualizaciones pendientes.
6. Considerar auditoría de seguridad externa.

---

### Checklist de recuperación

```
□ Backup disponible identificado y verificado
□ Nuevo servidor / entorno preparado
□ LAMP reinstalado y configurado
□ Bases de datos restauradas y verificadas
□ Archivos web restaurados con permisos correctos
□ SSH funcionando con clave pública
□ Firewall UFW activo con reglas correctas
□ Fail2ban activo
□ Backups automáticos programados y probados
□ Monitorización activa
□ Certificados SSL válidos
□ Sitio web accesible y funcional
□ Prueba de acceso a la aplicación web completa
□ Notificación al cliente del restablecimiento del servicio
```

---

## 9. Contactos y escalado

| Rol | Nombre | Contacto | Disponibilidad |
|---|---|---|---|
| Administrador técnico | — | — | L-V 9:00-18:00 |
| Soporte urgente | — | — | 24/7 |
| Proveedor VPS | — | — | 24/7 |
| Responsable cliente | — | — | — |

### Panel de control del proveedor VPS

- **URL:** ________________
- **Usuario:** ________________
- **Nota:** Las credenciales del panel están guardadas en el gestor de contraseñas de la empresa.

---

## 10. Registro de cambios

Documenta aquí todos los cambios significativos realizados en el servidor.

| Fecha | Cambio realizado | Responsable | Resultado |
|---|---|---|---|
| 2024-01-01 | Instalación inicial del sistema LAMP | Admin | ✅ OK |
| | | | |
| | | | |

---

> **Nota final:** Este documento debe mantenerse actualizado tras cada cambio significativo en la infraestructura. Se recomienda guardar una copia impresa en lugar seguro como parte del plan de contingencia.
