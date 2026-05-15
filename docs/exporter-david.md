# David Novillo — Blackbox Exporter

## 1. Identificación

**Nombre y apellidos:** David Novillo  
**Exporter:** Blackbox Exporter  
**Puerto:** 9115  
**Rol adicional:** Integrador (Prometheus + Grafana)

---

## 2. Qué hace este exporter

Blackbox Exporter es un exporter de Prometheus que realiza sondeos activos hacia endpoints externos: URLs HTTP/HTTPS, puertos TCP, pings ICMP y consultas DNS. A diferencia de Node Exporter o MySQL Exporter, que se instalan en la máquina a monitorizar, Blackbox puede ejecutarse en cualquier sitio y monitorizar servicios remotos desde fuera. En nuestro proyecto lo usamos para verificar que la aplicación web (puerto 3000 de VM1) está respondiendo correctamente desde la perspectiva de un usuario externo, midiendo su disponibilidad y tiempo de respuesta.

---

## 3. Cómo se instala

```bash
# 1. Descargar Blackbox Exporter
wget https://github.com/prometheus/blackbox_exporter/releases/download/v0.25.0/blackbox_exporter-0.25.0.linux-amd64.tar.gz

# 2. Descomprimir
tar xvfz blackbox_exporter-0.25.0.linux-amd64.tar.gz

# 3. Mover binario al sistema
sudo mv blackbox_exporter-0.25.0.linux-amd64/blackbox_exporter /usr/local/bin/

# 4. Crear usuario de sistema
sudo useradd -rs /bin/false blackbox_exporter

# 5. Crear directorio de configuración
sudo mkdir /etc/blackbox_exporter

# 6. Crear archivo de configuración
sudo tee /etc/blackbox_exporter/blackbox.yml > /dev/null <<EOF
modules:
  http_2xx:
    prober: http
    timeout: 5s
    http:
      valid_http_versions: ["HTTP/1.1", "HTTP/2.0"]
      valid_status_codes: []
      method: GET
      follow_redirects: true
  tcp_connect:
    prober: tcp
    timeout: 5s
EOF

# 7. Crear el servicio systemd
sudo tee /etc/systemd/system/blackbox_exporter.service > /dev/null <<EOF
[Unit]
Description=Prometheus Blackbox Exporter
After=network.target

[Service]
User=blackbox_exporter
Group=blackbox_exporter
Type=simple
ExecStart=/usr/local/bin/blackbox_exporter --config.file=/etc/blackbox_exporter/blackbox.yml
Restart=always

[Install]
WantedBy=multi-user.target
EOF

# 8. Arrancar y habilitar
sudo systemctl daemon-reload
sudo systemctl start blackbox_exporter
sudo systemctl enable blackbox_exporter

# 9. Verificar — prueba manual sobre la app
sudo systemctl status blackbox_exporter
curl 'http://localhost:9115/probe?target=http://192.168.1.40:3000&module=http_2xx'
```

---

## 4. Tres métricas clave

### `probe_success`
- **Tipo:** Gauge
- **Qué mide:** Indica si el sondeo al endpoint fue exitoso. Devuelve 1 si la app respondió correctamente y 0 si falló (timeout, error HTTP, conexión rechazada, etc.).
- **Por qué la elegí:** Es la métrica más importante de Blackbox Exporter. Con un solo número sabe si la app está disponible o no desde fuera. Es la base de cualquier alerta de disponibilidad.

### `probe_duration_seconds`
- **Tipo:** Gauge
- **Qué mide:** Tiempo total en segundos que tardó el sondeo completo en completarse, incluyendo resolución DNS, conexión TCP y tiempo de respuesta HTTP.
- **Por qué la elegí:** Mide la latencia real de la app tal como la experimenta un usuario. Aunque la app esté "up", si tarda 10 segundos en responder hay un problema de rendimiento. Esta métrica lo detecta.

### `probe_http_status_code`
- **Tipo:** Gauge
- **Qué mide:** Código de estado HTTP devuelto por la app en el último sondeo (200, 302, 404, 500, etc.).
- **Por qué la elegí:** Permite distinguir entre una caída total (probe_success = 0) y una respuesta errónea (por ejemplo, la app devuelve 500 pero sigue respondiendo). Un código 500 sostenido indica un error en la aplicación que hay que investigar aunque técnicamente esté "online".

---

## 5. Alerta propuesta

**Si `probe_success` vale 0 durante más de 2 minutos, avisar.**

En lenguaje natural: si la aplicación web deja de responder correctamente durante más de 2 minutos, lanzar alerta de disponibilidad.

**Justificación del umbral:** 2 minutos permite absorber reinicios breves del servidor Node.js (que pueden tardar unos segundos) sin disparar falsas alarmas. Sin embargo, es suficientemente corto para alertar antes de que los usuarios lleven demasiado tiempo sin poder acceder a la aplicación.

---

## 6. Limitación

Blackbox Exporter solo verifica que la app responde con el código HTTP correcto, pero no puede verificar que el contenido de la respuesta sea el esperado (por ejemplo, que la tabla de datos realmente tenga datos). Para eso habría que configurar módulos con `fail_if_body_not_matches_regexp`, lo que añade complejidad. Además, en aplicaciones que requieren autenticación para acceder al contenido real (como la nuestra, que tiene login), Blackbox solo puede verificar el endpoint público (`/login`) y no puede autenticarse para comprobar si `/tabla` funciona correctamente.
