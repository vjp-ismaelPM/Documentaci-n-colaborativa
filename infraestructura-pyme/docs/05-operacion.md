# 05 — Guía de Mantenimiento Diario

**Cliente:** PYME  
**Sistema:** Ubuntu Server 22.04 + Apache + PHP + MariaDB  
**Actualizado:** 2025

---

## 1. Comprobaciones diarias (5 minutos)

### Verificar que los servicios están activos

```bash
systemctl status apache2
systemctl status mariadb
systemctl status ssh
```

Si alguno aparece como `inactive` o `failed`, reiniciarlo:

```bash
systemctl restart apache2
systemctl restart mariadb
```

### Ver el uso de disco

```bash
df -h
```

Si alguna partición supera el **80%**, avisar al responsable técnico.

### Ver el uso de RAM y CPU

```bash
free -h
top -bn1 | head -20
```

---

## 2. Comprobaciones semanales

### Revisar logs de errores de Apache

```bash
tail -50 /var/log/apache2/error.log
```

Buscar líneas con `[error]` o `[crit]`. Si hay errores repetidos, anotarlos.

### Revisar intentos de acceso SSH fallidos

```bash
grep "Failed password" /var/log/auth.log | tail -20
```

Muchos intentos desde la misma IP pueden indicar un ataque. Bloquear con:

```bash
ufw deny from <IP_SOSPECHOSA>
```

### Actualizar el sistema

```bash
apt update && apt upgrade -y
```

Reiniciar si el sistema lo indica:

```bash
[ -f /var/run/reboot-required ] && echo "REINICIO NECESARIO"
```

---

## 3. Copias de seguridad

Las copias se ejecutan automáticamente cada noche a las **02:00** mediante cron.  
Comprobar que se han generado correctamente:

```bash
ls -lh /backups/db/
ls -lh /backups/files/
```

Debe haber al menos un archivo de las últimas 24 horas. Si no existe, ejecutar manualmente:

```bash
/usr/local/bin/backup.sh
```

---

## 4. Estado del firewall

```bash
ufw status verbose
```

Puertos permitidos habituales:

| Puerto | Servicio   |
|--------|------------|
| 22     | SSH        |
| 80     | HTTP       |
| 443    | HTTPS      |

No abrir otros puertos sin consultar al técnico.

---

## 5. Monitorización con Netdata

Si Netdata está instalado, acceder desde el navegador:

```
http://<IP_DEL_SERVIDOR>:19999
```

Revisar:
- CPU: no debe superar el 90% de forma sostenida.
- RAM: si queda menos de 200 MB libres, investigar qué proceso consume.
- Disco: alertas si supera el 80%.

---

## 6. Contacto de soporte

| Situación                  | Acción                          |
|----------------------------|---------------------------------|
| Servicio caído             | Reiniciar + notificar al técnico |
| Disco > 80%                | Notificar al técnico             |
| Backup no generado         | Ejecutar manual + notificar      |
| Intrusión sospechosa       | Bloquear IP + notificar urgente  |

**Técnico responsable:** ___________________  
**Teléfono / correo:** ___________________