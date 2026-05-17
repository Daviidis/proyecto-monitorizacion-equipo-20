# Exporter: Node Exporter

**Nombre y apellidos:** Henry Carrillo  
**Rol:** Técnico 1  
**Exporter:** Node Exporter  
**Puerto:** 9100  

---

## 1. Identificación

- **Nombre:** Henry Carrillo
- **Exporter trabajado:** Node Exporter (prometheus/node_exporter)
- **Versión instalada:** v1.8.2
- **Puerto de exposición:** 9100
- **Endpoint de métricas:** `http://<IP>:9100/metrics`

---

## 2. Qué hace este exporter

Node Exporter es un programa que se instala en el servidor Linux que quieres monitorizar y se encarga de recopilar datos sobre el estado de esa máquina: cuánta CPU está usando, cuánta memoria queda libre, qué espacio hay en disco y cuánto tráfico pasa por la red. Una vez recogidos, los expone en un formato que Prometheus entiende para ir a buscarlos cada cierto tiempo.

Lo que hace especial a Node Exporter es que no mide la aplicación que corre en el servidor, sino la salud del servidor en sí mismo. En nuestro proyecto, si la aplicación empieza a ir lenta, Node Exporter nos permite saber si el problema está en que el servidor está saturado de CPU o sin memoria, antes de buscar la causa en el código de la app.

A diferencia de otros exporters del proyecto como Blackbox, que mira la app desde fuera, Node Exporter mira el servidor desde dentro. Los dos son necesarios y se complementan.

---

## 3. Cómo se instala

Ejecutar en el servidor Ubuntu monitorizado como usuario con permisos sudo:

```bash
# 1. Descargar Node Exporter v1.8.2
wget https://github.com/prometheus/node_exporter/releases/download/v1.8.2/node_exporter-1.8.2.linux-amd64.tar.gz

# 2. Descomprimir
tar xvfz node_exporter-1.8.2.linux-amd64.tar.gz

# 3. Mover el binario al sistema
sudo mv node_exporter-1.8.2.linux-amd64/node_exporter /usr/local/bin/

# 4. Crear usuario de sistema sin shell (seguridad)
sudo useradd -rs /bin/false node_exporter

# 5. Crear el servicio systemd
sudo tee /etc/systemd/system/node_exporter.service << 'EOF'
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

# 6. Arrancar y habilitar el servicio
sudo systemctl daemon-reload
sudo systemctl start node_exporter
sudo systemctl enable node_exporter

# 7. Comprobar que está corriendo
sudo systemctl status node_exporter

# 8. Verificar que expone métricas
curl http://localhost:9100/metrics | head -20

# 9. Abrir el puerto en el firewall (si UFW está activo)
sudo ufw allow 9100/tcp
sudo ufw reload
```

---

## 4. Tres Métricas Clave

### Métrica 1: `node_cpu_seconds_total`

- **Tipo:** Counter
- **Qué mide:** El tiempo total acumulado (en segundos) que cada núcleo de CPU ha pasado en cada modo de funcionamiento: `user` (procesos de usuario), `system` (procesos del kernel), `idle` (sin hacer nada), `iowait` (esperando operaciones de disco), entre otros.
- **Por qué la he elegido:** Es la métrica fundamental para calcular el porcentaje de uso de CPU del servidor. En Grafana se usa la expresión `rate(node_cpu_seconds_total{mode="idle"}[5m])` para obtener el % de CPU libre y deducir cuánto está siendo usada. Si la app empieza a fallar, esta métrica nos dice si el servidor está saturado.

---

### Métrica 2: `node_memory_MemAvailable_bytes`

- **Tipo:** Gauge
- **Qué mide:** La cantidad de memoria RAM disponible en bytes en el momento actual. A diferencia de `MemFree`, esta métrica tiene en cuenta la memoria usada como caché que el sistema puede liberar si fuera necesario, por lo que es un valor más realista de la memoria realmente aprovechable.
- **Por qué la he elegido:** La memoria es uno de los recursos más críticos para la estabilidad de una aplicación web. Si `node_memory_MemAvailable_bytes` cae por debajo de un umbral, el sistema operativo empezará a usar swap y el rendimiento de la app se degradará drásticamente. Es la métrica de RAM que más información útil da de un vistazo.

---

### Métrica 3: `node_filesystem_avail_bytes`

- **Tipo:** Gauge
- **Qué mide:** El espacio libre disponible en bytes en cada sistema de ficheros montado en el servidor (partición raíz, `/var`, `/home`, etc.).
- **Por qué la he elegido:** Si el disco del servidor se llena, la aplicación deja de funcionar por completo: no puede escribir logs, la base de datos no puede guardar datos y el sistema operativo puede corromperse. Con esta métrica podemos anticipar ese problema y recibir una alerta antes de que ocurra. Se combina con `node_filesystem_size_bytes` para calcular el porcentaje de uso.

---

## 5. Una Alerta Propuesta

**En lenguaje natural:**  
"Si el espacio disponible en disco (`node_filesystem_avail_bytes`) baja por debajo del 15% del espacio total durante más de 5 minutos, enviar alerta de advertencia. Si baja por debajo del 5% durante más de 2 minutos, enviar alerta crítica."

**Justificación del umbral:**  
El 15% sirve como aviso temprano que da tiempo para actuar (limpiar logs, ampliar disco) sin urgencia. El 5% es el umbral crítico porque algunos sistemas operativos Linux reservan ese porcentaje para el usuario root, y por debajo de ese punto la aplicación puede empezar a fallar. Usar dos niveles permite distinguir entre una situación a vigilar y una emergencia real.

---

## 6. Una Limitación

Node Exporter no tiene ningún mecanismo de autenticación ni cifrado por defecto: cualquier persona que tenga acceso de red al puerto 9100 puede ver todas las métricas del servidor, incluyendo información sobre procesos, nombres de ficheros y configuración del sistema. En un entorno de producción real habría que configurar TLS y autenticación básica en el propio Node Exporter usando su opción `--web.config.file`, o proteger el puerto con un firewall que solo permita acceso desde la IP de Prometheus.
