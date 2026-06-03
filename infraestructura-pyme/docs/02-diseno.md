# 02 – Diseño de la Infraestructura

## Índice

- [02 – Diseño de la Infraestructura](#02--diseño-de-la-infraestructura)
  - [Índice](#índice)
  - [1. Diagrama de red](#1-diagrama-de-red)
  - [2. Descripción de componentes](#2-descripción-de-componentes)
  - [3. Tabla de versiones de software](#3-tabla-de-versiones-de-software)
  - [4. Esquema de red y puertos](#4-esquema-de-red-y-puertos)
    - [Puertos expuestos públicamente (desde Internet)](#puertos-expuestos-públicamente-desde-internet)
    - [Puertos internos (red local / VPN)](#puertos-internos-red-local--vpn)
    - [Reglas UFW resumen](#reglas-ufw-resumen)
  - [5. Diseño de bases de datos](#5-diseño-de-bases-de-datos)
    - [Base de datos `web_db` (aplicación web pública)](#base-de-datos-web_db-aplicación-web-pública)
    - [Base de datos `gestion_db` (gestión interna)](#base-de-datos-gestion_db-gestión-interna)
  - [6. Seguridad: zonas y accesos](#6-seguridad-zonas-y-accesos)
    - [Principios aplicados](#principios-aplicados)
    - [Archivos de configuración clave](#archivos-de-configuración-clave)

---

## 1. Diagrama de red

```
                          INTERNET
                              │
                    ┌─────────▼──────────┐
                    │   Balanceador      │
                    │   HAProxy          │
                    │   192.168.1.10     │
                    │   Puerto 80 / 443  │
                    └────────┬───────────┘
                             │  Round-robin
               ┌─────────────┴─────────────┐
               │                           │
   ┌───────────▼──────────┐   ┌────────────▼─────────┐
   │   Servidor Web 1     │   │   Servidor Web 2      │
   │   Apache + PHP       │   │   Apache + PHP        │
   │   192.168.1.20       │   │   192.168.1.21        │
   │   Puerto 80          │   │   Puerto 80           │
   └───────────┬──────────┘   └────────────┬──────────┘
               │                           │
               └─────────────┬─────────────┘
                             │  MySQL Protocol (3306)
                  ┌──────────▼──────────┐
                  │   Servidor BD       │
                  │   MySQL 8.0         │
                  │   192.168.1.30      │
                  │   Puerto 3306       │
                  └─────────────────────┘

   Acceso administración:
   ┌─────────────────────┐
   │   Sysadmin (Oficina)│──── SSH (22) ──► Todos los servidores
   │   192.168.1.0/24    │
   └─────────────────────┘
```

> **Nota:** En un despliegue inicial de bajo coste con un único servidor físico, los componentes HAProxy, Apache y MySQL pueden ejecutarse en la misma máquina, separando los servicios por puertos y usuarios del sistema. El diagrama anterior representa la arquitectura objetivo escalable.

---

## 2. Descripción de componentes

| Componente | Función | Observaciones |
|---|---|---|
| **HAProxy** | Balanceador de carga y punto de entrada HTTP/HTTPS | Distribuye tráfico entre los nodos Apache. Termina SSL (opcional). |
| **Apache 2.4** | Servidor web | Sirve las aplicaciones PHP. Configurado con VirtualHosts. |
| **PHP 8.2** | Motor de scripting | Integrado mediante `libapache2-mod-php`. |
| **MySQL 8.0** | Sistema gestor de base de datos relacional | Alberga la BD web y la BD de gestión interna. |
| **UFW** | Cortafuegos | Gestiona reglas de entrada/salida a nivel de host. |
| **Certbot** | Gestión automática de certificados TLS | Obtiene y renueva certificados Let's Encrypt. |
| **Netdata** | Monitorización en tiempo real | Panel web en puerto 19999 (acceso restringido). |
| **rsync + mysqldump** | Copias de seguridad | Ejecutados por cron. Rotación de 7 días. |

---

## 3. Tabla de versiones de software

| Software | Versión | Rol |
|---|---|---|
| Ubuntu Server | 22.04 LTS | Sistema operativo base |
| HAProxy | 2.6 | Balanceador de carga |
| Apache | 2.4.60 | Servidor web |
| PHP | 8.2 | Lenguaje de scripting |
| MySQL | 8.0 | Base de datos |
| Certbot | 2.9 | SSL/TLS automático |
| Netdata | 1.44 | Monitorización |
| UFW | 0.36.2 | Firewall |

---

## 4. Esquema de red y puertos

### Puertos expuestos públicamente (desde Internet)

| Puerto | Protocolo | Servicio | Destino |
|---|---|---|---|
| 80 | TCP | HTTP | HAProxy → Apache |
| 443 | TCP | HTTPS | HAProxy → Apache (TLS) |

### Puertos internos (red local / VPN)

| Puerto | Protocolo | Servicio | Acceso |
|---|---|---|---|
| 22 | TCP | SSH | Solo desde `192.168.1.0/24` |
| 3306 | TCP | MySQL | Solo desde servidores web |
| 19999 | TCP | Netdata | Solo desde `192.168.1.0/24` |

### Reglas UFW resumen

```bash
ufw default deny incoming
ufw default allow outgoing
ufw allow from 192.168.1.0/24 to any port 22    # SSH administración
ufw allow 80/tcp                                 # HTTP público
ufw allow 443/tcp                                # HTTPS público
ufw allow from 192.168.1.20 to any port 3306    # Web1 → BD
ufw allow from 192.168.1.21 to any port 3306    # Web2 → BD
ufw allow from 192.168.1.0/24 to any port 19999 # Netdata
ufw enable
```

---

## 5. Diseño de bases de datos

Se crearán dos bases de datos independientes en el mismo servidor MySQL:

### Base de datos `web_db` (aplicación web pública)

- **Usuario:** `web_user` (permisos solo sobre `web_db`)
- **Propósito:** Contenidos del sitio web (CMS, catálogo, usuarios públicos)

### Base de datos `gestion_db` (gestión interna)

- **Usuario:** `gestion_user` (permisos solo sobre `gestion_db`)
- **Propósito:** Facturación, inventario, empleados

```sql
-- Creación de bases de datos y usuarios
CREATE DATABASE web_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE DATABASE gestion_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

CREATE USER 'web_user'@'192.168.1.%' IDENTIFIED BY 'CAMBIAR_PASSWORD';
GRANT ALL PRIVILEGES ON web_db.* TO 'web_user'@'192.168.1.%';

CREATE USER 'gestion_user'@'192.168.1.%' IDENTIFIED BY 'CAMBIAR_PASSWORD';
GRANT ALL PRIVILEGES ON gestion_db.* TO 'gestion_user'@'192.168.1.%';

FLUSH PRIVILEGES;
```

> ⚠️ **Importante:** Sustituir `CAMBIAR_PASSWORD` por contraseñas seguras generadas con `openssl rand -base64 24` antes del despliegue en producción.

---

## 6. Seguridad: zonas y accesos

### Principios aplicados

- **Mínimo privilegio:** Cada usuario de base de datos solo accede a su esquema.
- **Defensa en profundidad:** UFW en cada host + red privada para servicios internos.
- **No exposición innecesaria:** MySQL y Netdata nunca accesibles desde Internet.
- **Cifrado en tránsito:** HTTPS obligatorio en producción (Certbot / Let's Encrypt).
- **Acceso SSH por clave:** Autenticación por contraseña deshabilitada en `/etc/ssh/sshd_config`.

### Archivos de configuración clave

| Archivo | Propósito |
|---|---|
| `/etc/haproxy/haproxy.cfg` | Configuración del balanceador |
| `/etc/apache2/sites-available/*.conf` | VirtualHosts de Apache |
| `/etc/mysql/mysql.conf.d/mysqld.cnf` | Configuración MySQL |
| `/etc/ssh/sshd_config` | Hardening SSH |
| `/etc/ufw/` | Reglas de firewall |

---

*Documento mantenido por el equipo de sistemas. Última actualización: sesión 4.*  
*Ver también: [01-analisis.md](01-analisis.md) | [03-planificacion.md](03-planificacion.md) | [servidor-web.md](04-instalacion/servidor-web.md)*