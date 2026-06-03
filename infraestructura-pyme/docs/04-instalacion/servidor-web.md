# Servidor Web: Apache + PHP

**Proyecto:** Infraestructura LAMP – PYME  
**Versión:** 1.0  
**Última actualización:** 2025-06  
**Responsable:** Documentalista de Plataforma

---

## Índice

1. [Descripción general](#1-descripción-general)
2. [Requisitos previos](#2-requisitos-previos)
3. [Instalación de Apache](#3-instalación-de-apache)
4. [Instalación de PHP](#4-instalación-de-php)
5. [Configuración de VirtualHosts](#5-configuración-de-virtualhosts)
6. [Módulos recomendados](#6-módulos-recomendados)
7. [Configuración de HAProxy (balanceador)](#7-configuración-de-haproxy-balanceador)
8. [Pruebas de funcionamiento](#8-pruebas-de-funcionamiento)
9. [Buenas prácticas de seguridad](#9-buenas-prácticas-de-seguridad)
10. [Resolución de problemas comunes](#10-resolución-de-problemas-comunes)

---

## 1. Descripción general

Este documento describe la instalación y configuración del servidor web **Apache 2.4** con soporte **PHP 8.1** sobre **Ubuntu Server 22.04 LTS**.

Apache actuará como servidor HTTP principal para:

- El sitio web público de la empresa (`www.empresa.local`)
- La aplicación de gestión interna (`gestion.empresa.local`)

```
[Internet] ──► [HAProxy :80/:443] ──► [Apache :8080]
                                           │
                                    [PHP-FPM 8.1]
                                           │
                                    [MySQL 8.0]
```

> **Nota:** Con HAProxy como balanceador frontal (sesión 4), Apache escucha en el puerto **8080** en lugar del 80.

---

## 2. Requisitos previos

| Elemento | Requisito |
|---|---|
| Sistema operativo | Ubuntu Server 22.04 LTS |
| RAM mínima | 1 GB (2 GB recomendado) |
| Disco | 20 GB libres en `/var` |
| Usuario | Con privilegios `sudo` |
| Red | IP estática configurada |
| Actualización del sistema | Aplicada antes de la instalación |

Antes de comenzar, actualizar el sistema:

```bash
sudo apt update && sudo apt upgrade -y
```

---

## 3. Instalación de Apache

### 3.1 Instalar el paquete

```bash
sudo apt install apache2 -y
```

### 3.2 Verificar estado del servicio

```bash
sudo systemctl status apache2
```

La salida esperada debe incluir `Active: active (running)`.

### 3.3 Habilitar inicio automático

```bash
sudo systemctl enable apache2
```

### 3.4 Comprobar la versión instalada

```bash
apache2 -v
# Server version: Apache/2.4.57 (Ubuntu)
```

### 3.5 Verificar desde el navegador

Acceder a `http://<IP_del_servidor>`. Debe aparecer la página de bienvenida de Apache.

---

## 4. Instalación de PHP

### 4.1 Instalar PHP 8.1 y extensiones necesarias

```bash
sudo apt install php8.1 php8.1-fpm php8.1-mysql php8.1-xml \
     php8.1-mbstring php8.1-curl php8.1-zip php8.1-gd -y
```

### 4.2 Verificar la versión de PHP

```bash
php -v
# PHP 8.1.x (cli)
```

### 4.3 Habilitar PHP-FPM y los módulos de Apache necesarios

```bash
sudo a2enmod proxy_fcgi setenvif
sudo a2enconf php8.1-fpm
sudo systemctl restart apache2
sudo systemctl enable php8.1-fpm
```

### 4.4 Comprobar PHP desde Apache

Crear un archivo de prueba:

```bash
sudo nano /var/www/html/info.php
```

Contenido:

```php
<?php phpinfo(); ?>
```

Acceder a `http://<IP>/info.php`. Debe mostrarse la página de información de PHP.

> ⚠️ **Importante:** Eliminar este archivo tras la verificación.

```bash
sudo rm /var/www/html/info.php
```

---

## 5. Configuración de VirtualHosts

### 5.1 Estructura de directorios

```bash
sudo mkdir -p /var/www/empresa/public_html
sudo mkdir -p /var/www/gestion/public_html
sudo chown -R www-data:www-data /var/www/empresa /var/www/gestion
sudo chmod -R 755 /var/www/empresa /var/www/gestion
```

### 5.2 VirtualHost para el sitio público

```bash
sudo nano /etc/apache2/sites-available/empresa.conf
```

```apache
<VirtualHost *:8080>
    ServerName www.empresa.local
    ServerAlias empresa.local
    DocumentRoot /var/www/empresa/public_html

    <Directory /var/www/empresa/public_html>
        Options -Indexes +FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    <FilesMatch \.php$>
        SetHandler "proxy:unix:/run/php/php8.1-fpm.sock|fcgi://localhost"
    </FilesMatch>

    ErrorLog ${APACHE_LOG_DIR}/empresa_error.log
    CustomLog ${APACHE_LOG_DIR}/empresa_access.log combined
</VirtualHost>
```

### 5.3 VirtualHost para la gestión interna

```bash
sudo nano /etc/apache2/sites-available/gestion.conf
```

```apache
<VirtualHost *:8080>
    ServerName gestion.empresa.local
    DocumentRoot /var/www/gestion/public_html

    <Directory /var/www/gestion/public_html>
        Options -Indexes +FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    <FilesMatch \.php$>
        SetHandler "proxy:unix:/run/php/php8.1-fpm.sock|fcgi://localhost"
    </FilesMatch>

    # Acceso restringido a la red interna
    <RequireAll>
        Require ip 192.168.1.0/24
    </RequireAll>

    ErrorLog ${APACHE_LOG_DIR}/gestion_error.log
    CustomLog ${APACHE_LOG_DIR}/gestion_access.log combined
</VirtualHost>
```

### 5.4 Activar los sitios y cambiar puerto de escucha

Cambiar el puerto de escucha (con HAProxy como frontal):

```bash
sudo nano /etc/apache2/ports.conf
```

Modificar:

```apache
Listen 8080
```

Activar los sitios y deshabilitar el default:

```bash
sudo a2ensite empresa.conf gestion.conf
sudo a2dissite 000-default.conf
sudo apache2ctl configtest   # debe devolver: Syntax OK
sudo systemctl reload apache2
```

---

## 6. Módulos recomendados

| Módulo | Comando de activación | Uso |
|---|---|---|
| `rewrite` | `sudo a2enmod rewrite` | URLs amigables (.htaccess) |
| `headers` | `sudo a2enmod headers` | Cabeceras de seguridad HTTP |
| `ssl` | `sudo a2enmod ssl` | HTTPS nativo (sin HAProxy) |
| `deflate` | `sudo a2enmod deflate` | Compresión de respuestas |
| `expires` | `sudo a2enmod expires` | Control de caché del navegador |

Activar los módulos necesarios:

```bash
sudo a2enmod rewrite headers deflate expires
sudo systemctl restart apache2
```

### 6.1 Cabeceras de seguridad HTTP

Añadir al VirtualHost o en un archivo de configuración global:

```apache
Header always set X-Content-Type-Options "nosniff"
Header always set X-Frame-Options "SAMEORIGIN"
Header always set X-XSS-Protection "1; mode=block"
Header always set Referrer-Policy "strict-origin-when-cross-origin"
```

---

## 7. Configuración de HAProxy (balanceador)

> Esta sección se añade en la **Sesión 4** del proyecto, a petición del cliente.  
> Issue relacionado: `#XX – Añadir balanceador HAProxy a la infraestructura`

### 7.1 Instalación de HAProxy

```bash
sudo apt install haproxy -y
haproxy -v
# HAProxy version 2.4.x
```

### 7.2 Configuración básica

```bash
sudo nano /etc/haproxy/haproxy.cfg
```

```haproxy
global
    log /dev/log local0
    maxconn 2000
    user haproxy
    group haproxy
    daemon

defaults
    log     global
    mode    http
    option  httplog
    option  dontlognull
    timeout connect 5s
    timeout client  30s
    timeout server  30s

frontend http_front
    bind *:80
    default_backend apache_back

backend apache_back
    balance roundrobin
    option httpchk GET /
    server web1 127.0.0.1:8080 check
```

### 7.3 Habilitar y verificar HAProxy

```bash
sudo systemctl enable haproxy
sudo systemctl start haproxy
sudo systemctl status haproxy
```

Verificar que HAProxy escucha en el puerto 80:

```bash
sudo ss -tlnp | grep ':80'
```

---

## 8. Pruebas de funcionamiento

### 8.1 Verificar que Apache responde

```bash
curl -I http://localhost:8080
# HTTP/1.1 200 OK
```

### 8.2 Verificar que HAProxy redirige correctamente

```bash
curl -I http://localhost
# HTTP/1.1 200 OK
```

### 8.3 Verificar procesamiento PHP

```bash
echo "<?php echo 'PHP OK'; ?>" | sudo tee /var/www/empresa/public_html/test.php
curl http://localhost/test.php
# PHP OK
sudo rm /var/www/empresa/public_html/test.php
```

### 8.4 Verificar logs de acceso

```bash
sudo tail -f /var/log/apache2/empresa_access.log
```

---

## 9. Buenas prácticas de seguridad

- **Ocultar la versión de Apache** para dificultar el fingerprinting:

```bash
sudo nano /etc/apache2/conf-available/security.conf
```

```apache
ServerTokens Prod
ServerSignature Off
```

- **Deshabilitar el listado de directorios** (`Options -Indexes` en todos los VirtualHosts).
- **Mantener Apache actualizado:**

```bash
sudo apt update && sudo apt upgrade apache2 -y
```

- **Revisar periódicamente los logs** de error y acceso en `/var/log/apache2/`.
- **Usar HTTPS** siempre en producción (ver [`ssh-firewall.md`](ssh-firewall.md) para la configuración del firewall asociada).

---

## 10. Resolución de problemas comunes

| Síntoma | Causa probable | Solución |
|---|---|---|
| Apache no arranca | Puerto 80/8080 ocupado | `sudo ss -tlnp \| grep 8080` y detener el proceso conflictivo |
| Error 403 Forbidden | Permisos incorrectos | `sudo chown -R www-data:www-data /var/www/` |
| PHP no se interpreta | Módulo PHP no activo | `sudo a2enconf php8.1-fpm && sudo systemctl restart apache2` |
| Error 502 Bad Gateway | PHP-FPM no está corriendo | `sudo systemctl start php8.1-fpm` |
| VirtualHost no responde | Sitio no activado | `sudo a2ensite nombre.conf && sudo systemctl reload apache2` |

---

## Referencias

- [Documentación oficial Apache 2.4](https://httpd.apache.org/docs/2.4/)
- [PHP 8.1 – php.net](https://www.php.net/releases/8.1/)
- [HAProxy Documentation](https://www.haproxy.org/#docs)
- → Ver también: [`base-de-datos.md`](base-de-datos.md) | [`ssh-firewall.md`](ssh-firewall.md)