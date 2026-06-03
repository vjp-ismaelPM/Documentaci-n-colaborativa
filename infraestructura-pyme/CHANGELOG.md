# CHANGELOG

Todos los cambios importantes del proyecto se documentan aquí.
Formato basado en [Keep a Changelog](https://keepachangelog.com/es/1.0.0/).

---

## [1.1.0] — 2025-06-03

### Añadido
- `05-operacion.md`: guía de mantenimiento diario del servidor LAMP.
- `06-recuperacion.md`: plan de recuperación ante desastres con 4 escenarios.

### Modificado
- Estructura de documentación ampliada con sección de operaciones.

---

## [1.0.0] — 2025-05-XX

### Añadido
- Configuración inicial del servidor Ubuntu Server 22.04.
- Instalación y configuración de Apache + PHP (LAMP).
- Instalación y configuración de MariaDB con dos bases de datos (`web_db`, `gestion_db`).
- Acceso remoto SSH con autenticación por clave.
- Configuración de firewall UFW (puertos 22, 80, 443).
- Script de copias de seguridad automáticas (`mysqldump` + `rsync`) con rotación de 7 días.
- Monitorización del sistema con Netdata.