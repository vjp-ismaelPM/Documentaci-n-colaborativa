# Base de Datos: MySQL / MariaDB

**Proyecto:** Infraestructura LAMP – PYME  
**Versión:** 1.0  
**Última actualización:** 2025-06  
**Responsable:** Documentalista de Plataforma

---

## Índice

1. [Descripción general](#1-descripción-general)
2. [Requisitos previos](#2-requisitos-previos)
3. [Instalación de MySQL 8.0](#3-instalación-de-mysql-80)
4. [Securización inicial](#4-securización-inicial)
5. [Creación de bases de datos y usuarios](#5-creación-de-bases-de-datos-y-usuarios)
6. [Configuración de rendimiento básica](#6-configuración-de-rendimiento-básica)
7. [Gestión del servicio](#7-gestión-del-servicio)
8. [Acceso remoto controlado](#8-acceso-remoto-controlado)
9. [Verificación de la instalación](#9-verificación-de-la-instalación)
10. [Buenas prácticas de seguridad](#10-buenas-prácticas-de-seguridad)
11. [Resolución de problemas comunes](#11-resolución-de-problemas-comunes)

---

## 1. Descripción

Este documento describe la instalación y configuración de **MySQL 8.0** como servidor de base de datos en **Ubuntu Server 22.04 LTS**.

Se crearán dos bases de datos independientes:

| Base de datos | Propósito | Usuario asociado |
|---|---|---|
| `db_web` | Datos del sitio web público | `usuario_web` |
| `db_gestion` | Aplicación de gestión interna | `usuario_gestion` |

Arquitectura de acceso:

```
[Apache/PHP] ──► [MySQL 8.0 :3306 (localhost)]
                       ├── db_web
                       └── db_gestion
```

> MySQL **no es accesible desde el exterior**. Solo acepta conexiones desde `localhost` (127.0.0.1). Las aplicaciones PHP se conectan localmente.

---

## 2. Requisitos previos

| Elemento | Requisito |
|---|---|
| Sistema operativo | Ubuntu Server 22.04 LTS |
| RAM mínima | 512 MB para MySQL (1 GB recomendado) |
| Disco | 10 GB libres en `/var/lib/mysql` |
| Apache + PHP instalados | Ver [`servidor-web.md`](servidor-web.md) |

Actualizar el sistema antes de instalar:

```bash
sudo apt update && sudo apt upgrade -y
```

---

## 3. Instalación de MySQL 8.0

### 3.1 Instalar el paquete

```bash
sudo apt install mysql-server -y
```

### 3.2 Verificar el estado del servicio

```bash
sudo systemctl status mysql
# Active: active (running)
```

### 3.3 Habilitar inicio automático

```bash
sudo systemctl enable mysql
```

### 3.4 Verificar la versión instalada

```bash
mysql --version
# mysql  Ver 8.0.x Distrib 8.0.x, for Linux (x86_64)
```

---

## 4. Securización inicial

Ejecutar el asistente de seguridad incluido con MySQL:

```bash
sudo mysql_secure_installation
```

Respuestas recomendadas durante el asistente:

| Pregunta | Respuesta recomendada |
|---|---|
| ¿Configurar VALIDATE PASSWORD component? | `Y` |
| Nivel de política de contraseñas | `2` (STRONG) |
| ¿Cambiar contraseña de root? | `Y` (establecer una contraseña robusta) |
| ¿Eliminar usuarios anónimos? | `Y` |
| ¿Deshabilitar login de root remoto? | `Y` |
| ¿Eliminar base de datos de prueba? | `Y` |
| ¿Recargar tablas de privilegios? | `Y` |

> ⚠️ **Guardar la contraseña de root en un gestor de contraseñas seguro.** No se podrá recuperar sin intervención en el sistema.

### 4.1 Acceso inicial como root

En Ubuntu 22.04, MySQL usa autenticación por socket para root por defecto:

```bash
sudo mysql
```

Para habilitar autenticación con contraseña si se necesita:

```sql
ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'ContraseñaSegura123!';
FLUSH PRIVILEGES;
```

---

## 5. Creación de bases de datos y usuarios

Conectarse al servidor:

```bash
sudo mysql -u root -p
```

### 5.1 Crear la base de datos para el sitio web

```sql
CREATE DATABASE db_web
    CHARACTER SET utf8mb4
    COLLATE utf8mb4_unicode_ci;
```

### 5.2 Crear el usuario para la web

```sql
CREATE USER 'usuario_web'@'localhost' IDENTIFIED BY 'Web_Pass_2024!';
GRANT SELECT, INSERT, UPDATE, DELETE ON db_web.* TO 'usuario_web'@'localhost';
FLUSH PRIVILEGES;
```

### 5.3 Crear la base de datos de gestión interna

```sql
CREATE DATABASE db_gestion
    CHARACTER SET utf8mb4
    COLLATE utf8mb4_unicode_ci;
```

### 5.4 Crear el usuario para la gestión

```sql
CREATE USER 'usuario_gestion'@'localhost' IDENTIFIED BY 'Gestion_Pass_2024!';
GRANT SELECT, INSERT, UPDATE, DELETE, CREATE, ALTER ON db_gestion.* TO 'usuario_gestion'@'localhost';
FLUSH PRIVILEGES;
```

### 5.5 Verificar los usuarios creados

```sql
SELECT user, host, plugin FROM mysql.user;
```

Salida esperada (entre otros):

```
+------------------+-----------+-----------------------+
| user             | host      | plugin                |
+------------------+-----------+-----------------------+
| usuario_web      | localhost | caching_sha2_password |
| usuario_gestion  | localhost | caching_sha2_password |
| root             | localhost | auth_socket           |
+------------------+-----------+-----------------------+
```

### 5.6 Verificar los privilegios

```sql
SHOW GRANTS FOR 'usuario_web'@'localhost';
SHOW GRANTS FOR 'usuario_gestion'@'localhost';
```

---

## 6. Configuración de rendimiento básica

Editar el archivo de configuración principal:

```bash
sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
```

Ajustes recomendados para un servidor pequeño (2 GB RAM):

```ini
[mysqld]
# Bind solo a localhost (seguridad)
bind-address            = 127.0.0.1

# Caché de consultas (tamaño según RAM disponible)
innodb_buffer_pool_size = 512M

# Logs de consultas lentas (útil para optimización)
slow_query_log          = 1
slow_query_log_file     = /var/log/mysql/mysql-slow.log
long_query_time         = 2

# Límite de conexiones
max_connections         = 100

# Zona horaria del servidor
default_time_zone       = '+01:00'

# Juego de caracteres por defecto
character-set-server    = utf8mb4
collation-server        = utf8mb4_unicode_ci
```

Aplicar los cambios:

```bash
sudo systemctl restart mysql
sudo systemctl status mysql
```

---

## 7. Gestión del servicio

| Acción | Comando |
|---|---|
| Iniciar MySQL | `sudo systemctl start mysql` |
| Detener MySQL | `sudo systemctl stop mysql` |
| Reiniciar MySQL | `sudo systemctl restart mysql` |
| Recargar configuración | `sudo systemctl reload mysql` |
| Ver estado | `sudo systemctl status mysql` |
| Ver logs en tiempo real | `sudo journalctl -u mysql -f` |

---

## 8. Acceso remoto controlado

Por defecto, MySQL solo escucha en `localhost`. Si en el futuro fuera necesario acceso remoto desde una IP concreta de la red interna (ej. una herramienta de administración), seguir estos pasos:

### 8.1 Crear usuario con acceso remoto restringido

```sql
CREATE USER 'admin_remoto'@'192.168.1.10' IDENTIFIED BY 'Admin_Remote_2024!';
GRANT SELECT ON *.* TO 'admin_remoto'@'192.168.1.10';
FLUSH PRIVILEGES;
```

### 8.2 Ajustar bind-address para red interna

```ini
# En /etc/mysql/mysql.conf.d/mysqld.cnf
bind-address = 192.168.1.X   # IP del servidor en la red interna
```

### 8.3 Abrir el puerto en el firewall

```bash
sudo ufw allow from 192.168.1.10 to any port 3306
```

> ⚠️ **Nunca exponer el puerto 3306 a Internet.** Ver [`ssh-firewall.md`](ssh-firewall.md).

---

## 9. Verificación de la instalación

### 9.1 Conectarse con el usuario de la web

```bash
mysql -u usuario_web -p db_web
```

### 9.2 Crear una tabla de prueba

```sql
CREATE TABLE prueba (
    id INT AUTO_INCREMENT PRIMARY KEY,
    nombre VARCHAR(100) NOT NULL,
    creado_en TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO prueba (nombre) VALUES ('Prueba de conexión');
SELECT * FROM prueba;
```

Salida esperada:

```
+----+--------------------+---------------------+
| id | nombre             | creado_en           |
+----+--------------------+---------------------+
|  1 | Prueba de conexión | 2025-06-01 10:00:00 |
+----+--------------------+---------------------+
```

### 9.3 Limpiar y salir

```sql
DROP TABLE prueba;
EXIT;
```

---

## 10. Buenas prácticas de seguridad

- **Principio de mínimo privilegio:** cada usuario solo tiene los permisos estrictamente necesarios para su aplicación.
- **Contraseñas robustas:** mínimo 12 caracteres, combinando mayúsculas, minúsculas, números y símbolos.
- **Sin acceso root remoto:** el usuario `root` de MySQL nunca debe poder conectarse desde fuera de `localhost`.
- **Copias de seguridad regulares:** ver [`backups.md`](../04-instalacion/backups.md) para la estrategia con `mysqldump`.
- **Actualizaciones periódicas:**

```bash
sudo apt update && sudo apt upgrade mysql-server -y
```

- **Monitorizar el log de consultas lentas** en `/var/log/mysql/mysql-slow.log` para detectar problemas de rendimiento.

---

## 11. Resolución de problemas comunes

| Síntoma | Causa probable | Solución |
|---|---|---|
| `ERROR 1045: Access denied` | Contraseña incorrecta o usuario inexistente | Verificar usuario con `SELECT user, host FROM mysql.user;` |
| `ERROR 2002: Can't connect to socket` | MySQL no está corriendo | `sudo systemctl start mysql` |
| `ERROR 1044: Access denied to database` | Privilegios insuficientes | `SHOW GRANTS FOR 'usuario'@'host';` y reasignar |
| MySQL no arranca tras cambio de config | Error de sintaxis en `.cnf` | `sudo mysqld --validate-config` |
| Disco lleno en `/var/lib/mysql` | Logs binarios acumulados | `sudo mysqlcheck --all-databases` y purgar logs |

---

## Referencias

- [Documentación oficial MySQL 8.0](https://dev.mysql.com/doc/refman/8.0/en/)
- [Ubuntu Server Guide – MySQL](https://ubuntu.com/server/docs/databases-mysql)
- → Ver también: [`servidor-web.md`](servidor-web.md) | [`backups.md`](backups.md) | [`ssh-firewall.md`](ssh-firewall.md)