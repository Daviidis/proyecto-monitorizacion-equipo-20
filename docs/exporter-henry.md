# Henry Carrillo — Node Exporter

## 1. Identificación

**Nombre y apellidos:** Henry Carrillo  
**Exporter:** Node Exporter  
**Puerto:** 9100

---

## 2. Qué hace este exporter

Node Exporter es el exporter oficial de Prometheus para sistemas Linux. Se instala directamente en el servidor que queremos monitorizar y expone métricas del hardware y del sistema operativo: uso de CPU, memoria RAM, espacio en disco, tráfico de red y carga del sistema. En nuestro proyecto monitoriza la VM1 (192.168.1.40), que es el servidor Ubuntu donde está desplegada la aplicación web. Sin Node Exporter no tendríamos visibilidad sobre si el servidor tiene recursos suficientes para mantener la app funcionando.

---

## 3. Cómo se instala

```bash
# 1. Descargar Node Exporter
wget https://github.com/prometheus/node_exporter/releases/download/v1.8.2/node_exporter-1.8.2.linux-amd64.tar.gz

# 2. Descomprimir
tar xvfz node_exporter-1.8.2.linux-amd64.tar.gz

# 3. Mover binario al sistema
sudo mv node_exporter-1.8.2.linux-amd64/node_exporter /usr/local/bin/

# 4. Crear usuario de sistema sin shell
sudo useradd -rs /bin/false node_exporter

# 5. Crear el servicio systemd
sudo tee /etc/systemd/system/node_exporter.service > /dev/null <<EOF
[Unit]
Description=Prometheus Node Exporter
After=network.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter
Restart=always

[Install]
WantedBy=multi-user.target
EOF

# 6. Arrancar y habilitar
sudo systemctl daemon-reload
sudo systemctl start node_exporter
sudo systemctl enable node_exporter

# 7. Verificar
sudo systemctl status node_exporter
curl http://localhost:9100/metrics | head -20
```

---

## 4. Tres métricas clave

### `node_cpu_seconds_total`
- **Tipo:** Counter
- **Qué mide:** Tiempo acumulado que la CPU ha pasado en cada modo (user, system, idle, iowait). Aplicando `rate()` sobre el modo idle se puede calcular el porcentaje de uso de CPU.
- **Por qué la elegí:** Es la métrica más fundamental para saber si el servidor está saturado. Un uso de CPU sostenido por encima del 80% indica que la app o algún proceso está consumiendo demasiados recursos.

### `node_memory_MemAvailable_bytes`
- **Tipo:** Gauge
- **Qué mide:** Memoria RAM disponible en el sistema en bytes. Combinada con `node_memory_MemTotal_bytes` permite calcular el porcentaje de RAM utilizada.
- **Por qué la elegí:** Si la memoria se agota, el sistema empieza a usar swap y la aplicación se vuelve muy lenta o directamente cae. Es imprescindible para anticipar este problema.

### `node_filesystem_avail_bytes`
- **Tipo:** Gauge
- **Qué mide:** Espacio disponible en cada sistema de ficheros montado. Se combina con `node_filesystem_size_bytes` para calcular el porcentaje de uso del disco.
- **Por qué la elegí:** La base de datos MariaDB y los logs de la aplicación escriben en disco continuamente. Si el disco se llena, la base de datos deja de funcionar y la app cae. Esta métrica es crítica para evitar ese escenario.

---

## 5. Alerta propuesta

**Si el uso de CPU supera el 85% durante más de 5 minutos, avisar.**

En lenguaje natural: si `node_cpu_seconds_total` (calculado como porcentaje de CPU no idle) supera el 85% de media durante 5 minutos, lanzar alerta.

**Justificación del umbral:** Un 85% sostenido durante 5 minutos indica que el servidor está bajo carga real y no simplemente un pico puntual. Por debajo de ese umbral, picos cortos son normales (arranque de procesos, compilaciones, etc.). Por encima de ese umbral durante ese tiempo hay riesgo de que la app empiece a dar timeouts o errores.

---

## 6. Limitación

Node Exporter expone métricas del sistema operativo pero no tiene visibilidad sobre la aplicación en sí. No puede decirte si la app está respondiendo correctamente ni cuántas peticiones está procesando — para eso se necesita el Blackbox Exporter o métricas propias de la app. Además, en sistemas con muchos discos o interfaces de red, la cantidad de series temporales puede crecer mucho y consumir más recursos de los esperados en Prometheus.
