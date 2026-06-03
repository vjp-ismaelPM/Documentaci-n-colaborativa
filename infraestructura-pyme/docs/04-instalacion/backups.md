# Copias de seguridad

La estrategia de backups se basa en `mysqldump` para exportar las bases de datos y `rsync` para copiarlas a un servidor remoto. La automatización se hace con `cron`.

## 1. Script de backup

Creamos el script en `/usr/local/bin/backup.sh`:

```bash
#!/bin/bash

FECHA=$(date +%Y%m%d)
DESTINO="/var/backups/mysql"

mkdir -p "$DESTINO"

# Volcado de las bases de datos
mysqldump -u root -p'contraseña' db_web > "$DESTINO/db_web_$FECHA.sql"
mysqldump -u root -p'contraseña' db_gestion > "$DESTINO/db_gestion_$FECHA.sql"

# Copia al servidor remoto
rsync -avz "$DESTINO/" usuario@servidor-remoto:/backups/

# Eliminar backups de más de 7 días
find "$DESTINO" -name "*.sql" -mtime +7 -delete
```

Le damos permisos de ejecución:

```bash
sudo chmod +x /usr/local/bin/backup.sh
```

## 2. Automatización con cron

```bash
sudo crontab -e
```

Añadimos la línea para que se ejecute cada noche a las 2:00:

```
0 2 * * * /usr/local/bin/backup.sh >> /var/log/backup.log 2>&1
```

## 3. Restaurar una copia de seguridad

```bash
mysql -u root -p db_web < /var/backups/mysql/db_web_20240601.sql
```

## 4. Verificación

Para comprobar que el script funciona correctamente antes de dejarlo en producción:

```bash
sudo /usr/local/bin/backup.sh
ls -lh /var/backups/mysql/
tail -20 /var/log/backup.log
```

---

_Documento elaborado por Aitor Trujillo Pablo – Sesión 1_