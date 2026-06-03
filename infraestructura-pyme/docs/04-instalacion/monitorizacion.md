# Monitorización del sistema con Netdata

Netdata es una herramienta open source de monitorización en tiempo real. Muestra métricas del sistema a través de un panel web sin necesidad de configuración compleja.

## 1. Instalación

```bash
bash <(curl -Ss https://my-netdata.io/kickstart.sh)
```

Una vez instalado, habilitamos el inicio automático:

```bash
sudo systemctl enable netdata
sudo systemctl start netdata
```

## 2. Acceso al panel

El panel web está disponible en:

```
http://IP-DEL-SERVIDOR:19999
```

Para comprobar que el servicio está corriendo:

```bash
sudo systemctl status netdata
```

## 3. Métricas disponibles

Netdata recoge automáticamente métricas de CPU, memoria RAM, uso de disco, tráfico de red y estado de procesos. También detecta Apache y MySQL si están instalados y activa sus plugins de forma automática.

Para que funcione el plugin de Apache hay que tener habilitado `mod_status`:

```bash
sudo a2enmod status
sudo systemctl restart apache2
```

## 4. Alertas

Netdata incluye alertas predefinidas para los umbrales más comunes (CPU alta, poco espacio en disco, etc.). Se pueden personalizar en `/etc/netdata/health.d/`.

Para ver las alertas activas desde el panel, ir a la sección **Alerts** en el menú lateral.

---

_Documento elaborado por Aitor Trujillo Pablo – Sesión 1_