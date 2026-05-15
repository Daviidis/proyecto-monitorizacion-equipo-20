# Álvaro Oliva — MySQL Exporter

## 1. Identificación

**Nombre y apellidos:** Álvaro Oliva  
**Exporter:** MySQL Exporter (mysqld_exporter)  
**Puerto:** 9104

---

## 2. Qué hace este exporter

MySQL Exporter es el exporter oficial de Prometheus para bases de datos MySQL y MariaDB. Se conecta a la base de datos usando un usuario con permisos de solo lectura y expone métricas sobre el estado del motor: conexiones activas, consultas ejecutadas, consultas lentas, errores y operaciones de lectura/escritura. En nuestro proyecto la aplicación web usa una base de datos MariaDB con la tabla de ensamblaje de aviones A320, por lo que este exporter nos permite detectar si la base de datos está saturada o tiene problemas antes de que afecten a los usuarios.

---

## 3. Cómo se instala

```bash
# PASO PREVIO: Crear usuario MySQL para el exporter
sudo mysql -u root

# Dentro de MySQL/MariaDB:
CREATE USER 'prometheus_exporter'@'localhost' IDENTIFIED BY 'password_seguro' WITH MAX_USER_CONNECTIONS 3;
GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'prometheus_exporter'@'localhost';
FLUSH PRIVILEGES;
EXIT;

# 1. Descargar MySQL Exporter
wget https://github.com/prometheus/mysqld_exporter/releases/download/v0.15.1/mysqld_exporter-0.15.1.linux-amd64.tar.gz

# 2. Descomprimir
tar xvfz mysqld_exporter-0.15.1.linux-amd64.tar.gz

# 3. Mover binario al sistema
sudo mv mysqld_exporter-0.15.1.linux-amd64/mysqld_exporter /usr/local/bin/

# 4. Crear usuario de sistema
sudo useradd -rs /bin/false mysqld_exporter

# 5. Crear archivo de credenciales
sudo tee /etc/.mysqld_exporter.cnf > /dev/null <<EOF
[client]
user=prometheus_exporter
password=password_seguro
host=localhost
EOF

sudo chmod 600 /etc/.mysqld_exporter.cnf
sudo chown mysqld_exporter:mysqld_exporter /etc/.mysqld_exporter.cnf

# 6. Crear el servicio systemd
sudo tee /etc/systemd/system/mysqld_exporter.service > /dev/null <<EOF
[Unit]
Description=Prometheus MySQL Exporter
After=network.target
After=mysqld.service

[Service]
User=mysqld_exporter
Group=mysqld_exporter
Type=simple
ExecStart=/usr/local/bin/mysqld_exporter --config.my-cnf=/etc/.mysqld_exporter.cnf
Restart=always

[Install]
WantedBy=multi-user.target
EOF

# 7. Arrancar y habilitar
sudo systemctl daemon-reload
sudo systemctl start mysqld_exporter
sudo systemctl enable mysqld_exporter

# 8. Verificar
sudo systemctl status mysqld_exporter
curl http://localhost:9104/metrics | grep mysql_up
```

---

## 4. Tres métricas clave

### `mysql_up`
- **Tipo:** Gauge
- **Qué mide:** Indica si la conexión con la base de datos es exitosa. Devuelve 1 si MariaDB está accesible y 0 si hay un error de conexión.
- **Por qué la elegí:** Es la métrica más crítica. Si esta métrica vale 0, la aplicación web no puede consultar ni escribir datos, lo que significa que está completamente rota. Cualquier sistema de alertas debe empezar por aquí.

### `mysql_global_status_threads_connected`
- **Tipo:** Gauge
- **Qué mide:** Número de clientes actualmente conectados a MariaDB. Si este valor se acerca al máximo permitido (`mysql_global_variables_max_connections`), la base de datos empieza a rechazar nuevas conexiones.
- **Por qué la elegí:** En una app web con muchos usuarios simultáneos, agotar el pool de conexiones es un fallo común. Esta métrica permite detectarlo antes de que ocurra y ajustar la configuración o escalar.

### `mysql_global_status_slow_queries`
- **Tipo:** Counter
- **Qué mide:** Número acumulado de consultas que han tardado más de lo configurado en `long_query_time`. Un incremento sostenido indica que hay consultas mal optimizadas o que la base de datos está bajo demasiada carga.
- **Por qué la elegí:** Las consultas lentas degradan directamente la experiencia de usuario. Si la tabla de la app tarda mucho en cargarse, probablemente sea por consultas lentas en MariaDB. Esta métrica nos lo confirma.

---

## 5. Alerta propuesta

**Si `mysql_up` vale 0 durante más de 1 minuto, avisar de forma inmediata.**

En lenguaje natural: si la base de datos MariaDB deja de responder durante más de 1 minuto, lanzar alerta crítica.

**Justificación del umbral:** 1 minuto es suficiente para descartar reinicios rápidos del servicio (que pueden durar unos segundos) pero lo suficientemente corto para alertar antes de que los usuarios lleven mucho tiempo sin poder usar la app. Una caída de base de datos de más de 1 minuto debe ser investigada siempre.

---

## 6. Limitación

MySQL Exporter requiere que el usuario de base de datos tenga permisos específicos (`PROCESS`, `REPLICATION CLIENT`, `SELECT`). En entornos con políticas de seguridad muy estrictas, puede ser difícil obtener esos permisos. Además, algunas métricas avanzadas como las de rendimiento de InnoDB por tabla requieren activar el `performance_schema` en MariaDB, que no siempre está habilitado por defecto y tiene un pequeño impacto en el rendimiento del servidor de base de datos.
