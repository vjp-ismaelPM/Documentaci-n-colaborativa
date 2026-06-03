# 06 — Plan de Recuperación ante Desastres

**Cliente:** PYME  
**Sistema:** Ubuntu Server 22.04 + Apache + PHP + MariaDB  
**Actualizado:** 2025

---

## 1. Objetivos

| Concepto | Valor objetivo |
|----------|---------------|
| RPO (pérdida máxima de datos aceptable) | 24 horas |
| RTO (tiempo máximo para restaurar el servicio) | 4 horas |

---

## 2. Escenarios y respuesta

### Escenario A — Un servicio caído (Apache o MariaDB)

**Síntoma:** La web no carga o da error de conexión a base de datos.

**Pasos:**

```bash
# 1. Identificar qué servicio falló
systemctl status apache2
systemctl status mariadb

# 2. Reiniciar el servicio
systemctl restart apache2
systemctl restart mariadb

# 3. Comprobar logs si el reinicio falla
journalctl -u apache2 --since "10 minutes ago"
journalctl -u mariadb --since "10 minutes ago"
```

Tiempo estimado: **10–20 minutos.**

---

### Escenario B — Base de datos corrupta o borrada accidentalmente

**Pasos:**

```bash
# 1. Localizar el backup más reciente
ls -lht /backups/db/

# 2. Restaurar la base de datos afectada (ejemplo: web_db)
mysql -u root -p web_db < /backups/db/web_db_YYYYMMDD.sql

# 3. Verificar que la web funciona correctamente
systemctl restart apache2
```

Tiempo estimado: **30–60 minutos** según tamaño de la base de datos.

---

### Escenario C — Servidor inaccesible (no responde SSH ni web)

**Pasos:**

1. Acceder a la consola física o consola KVM/VPS del proveedor.
2. Comprobar si el servidor arrancó correctamente:
   ```bash
   dmesg | tail -30
   ```
3. Si hay fallo de disco, contactar al proveedor de hosting.
4. Si el sistema arranca pero SSH no responde:
   ```bash
   systemctl restart ssh
   ufw status   # asegurarse de que el puerto 22 está abierto
   ```

Tiempo estimado: **1–2 horas.**

---

### Escenario D — Servidor destruido o irrecuperable

**Pasos para restauración completa:**

1. Provisionar nuevo servidor Ubuntu Server 22.04.
2. Instalar el stack LAMP:
   ```bash
   apt update && apt install -y apache2 php libapache2-mod-php mariadb-server
   ```
3. Copiar los archivos web desde el backup:
   ```bash
   rsync -av /backups/files/ /var/www/html/
   ```
4. Restaurar ambas bases de datos:
   ```bash
   mysql -u root -p web_db        < /backups/db/web_db_ULTIMO.sql
   mysql -u root -p gestion_db    < /backups/db/gestion_db_ULTIMO.sql
   ```
5. Configurar UFW:
   ```bash
   ufw allow 22
   ufw allow 80
   ufw allow 443
   ufw enable
   ```
6. Verificar que la web es accesible.

Tiempo estimado: **2–4 horas.**

---

## 3. Localización de los backups

| Tipo          | Ruta local           | Retención |
|---------------|----------------------|-----------|
| Bases de datos | `/backups/db/`       | 7 días    |
| Archivos web  | `/backups/files/`    | 7 días    |
| Backup remoto | Servidor externo / NAS | 30 días |

Los backups remotos se sincronizan con `rsync` automáticamente tras cada copia local.

---

## 4. Lista de verificación post-restauración

Antes de dar el servicio por restaurado, confirmar:

- [ ] Apache responde en el puerto 80/443.
- [ ] La web carga correctamente en el navegador.
- [ ] La base de datos web contiene datos recientes.
- [ ] La base de datos de gestión interna es accesible.
- [ ] SSH funciona desde la red habitual.
- [ ] El firewall UFW está activo con las reglas correctas.
- [ ] El cron de backups está activo (`crontab -l`).
- [ ] Netdata (monitorización) está accesible.

---

## 5. Contactos de emergencia

| Rol                    | Nombre | Contacto |
|------------------------|--------|----------|
| Técnico responsable    |        |          |
| Proveedor de hosting   |        |          |
| Responsable del cliente|        |          |

---

## 6. Registro de incidencias

Anotar cada incidencia para mejorar el plan:

| Fecha | Escenario | Causa | Tiempo de resolución | Observaciones |
|-------|-----------|-------|----------------------|---------------|
|       |           |       |                      |               |