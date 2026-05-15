# Proyecto Monitorización — Prometheus + Grafana

## 1. Título y Equipo

**Nombre del proyecto:** Monitorización de aplicación web con Prometheus y Grafana

| Miembro | Rol | Exporter |
|---|---|---|
| Henry Carrillo | Técnico 1 | Node Exporter |
| Álvaro Oliva | Técnico 2 | MySQL Exporter |
| David Novillo | Técnico 3 + Integrador | Blackbox Exporter, Prometheus, Grafana |

---

## 2. Descripción de la aplicación monitorizada

La aplicación es un servidor web desarrollado con Node.js y Express que incluye un sistema de login con autenticación mediante sesiones y contraseñas hasheadas con bcrypt. Una vez autenticado, el usuario accede a una tabla con el estado de fabricación de aviones Airbus A320 en la Final Assembly Line (FAL). La app expone dos endpoints principales: `/login` para la autenticación y `/tabla` para la visualización de datos. Está desplegada sobre Ubuntu Server en el puerto 3000 con proxy inverso Nginx y utiliza MariaDB como base de datos. En producción serviría como panel de seguimiento de producción industrial en tiempo real.

---

## 3. Arquitectura

```
App Web (Node.js + MariaDB)
        |
        |-- Node Exporter     (puerto 9100)  →  métricas del SO
        |-- MySQL Exporter    (puerto 9104)  →  métricas de MariaDB
        |
        ↓
   Prometheus Server          (puerto 9090)  →  recolección y almacenamiento
        |
        |-- Blackbox Exporter (puerto 9115)  →  sondeos HTTP a la app
        ↓
      Grafana                 (puerto 3000)  →  visualización y alertas
```

**VM1** `192.168.1.40` — App + Node Exporter + MySQL Exporter  
**VM2** `192.168.1.20` — Prometheus + Blackbox Exporter + Grafana

---

## 4. Cómo se levanta el sistema

### VM1 — Levantar la aplicación

```bash
cd ~/proyecto-login
npm start &
```

### VM1 — Verificar que los exporters están activos

```bash
sudo systemctl status node_exporter
sudo systemctl status mysqld_exporter
```

Si no estuvieran activos:

```bash
sudo systemctl start node_exporter
sudo systemctl start mysqld_exporter
```

### VM2 — Verificar que Prometheus, Blackbox y Grafana están activos

```bash
sudo systemctl status prometheus
sudo systemctl status blackbox_exporter
sudo systemctl status grafana-server
```

Si no estuvieran activos:

```bash
sudo systemctl start prometheus
sudo systemctl start blackbox_exporter
sudo systemctl start grafana-server
```

### Acceso a las interfaces

- Prometheus: `http://192.168.1.20:9090`
- Grafana: `http://192.168.1.20:3000` (usuario: `admin`)
- Targets: `http://192.168.1.20:9090/targets`

---

## 5. Decisiones que hemos tomado

**¿Por qué estos exporters y no otros?**  
Node Exporter es el estándar para monitorizar el sistema operativo Linux donde corre la app. MySQL Exporter (mysqld_exporter) es el exporter oficial para MariaDB/MySQL, que es la base de datos que usa nuestra aplicación. Blackbox Exporter nos permite verificar la disponibilidad real de la app desde fuera, respondiendo a la pregunta "¿la app responde correctamente?" que los otros exporters no cubren.

**¿Por qué esos puertos?**  
Usamos los puertos por defecto de cada exporter (9100, 9104, 9115) para seguir las convenciones de Prometheus y facilitar la integración con dashboards estándar de Grafana.

**¿Qué descartasteis y por qué?**  
Descartamos el uso de cAdvisor (para contenedores Docker) porque la app está desplegada en modo tradicional con `npm start`, no con Docker Compose. También descartamos el Nginx Exporter porque Nginx actúa solo como proxy y no es el componente crítico a monitorizar.

---

## 6. Limitaciones conocidas

- El warning de diferencia de tiempo entre el navegador y Prometheus (`time drift`) aparece en la interfaz gráfica porque las VMs no tienen NTP perfectamente sincronizado. No afecta al funcionamiento pero puede causar pequeñas desviaciones en los resultados de algunas queries.
- Grafana se expone en el puerto 3000, el mismo que la app monitorizada. En producción habría que cambiar el puerto de Grafana en `/etc/grafana/grafana.ini` (parámetro `http_port`) para evitar conflictos si ambos servicios corrieran en la misma máquina.
- Las credenciales del MySQL Exporter (`password_seguro`) son de ejemplo y habría que cambiarlas por unas seguras en un entorno real.
- No se han configurado reglas de alertas formales en Prometheus (`alerting_rules.yml`). Las alertas descritas en la documentación de cada exporter están propuestas pero no implementadas.

---

## Fuentes consultadas

- Documentación oficial de Prometheus: https://prometheus.io/docs/
- Repositorio Node Exporter: https://github.com/prometheus/node_exporter
- Repositorio Blackbox Exporter: https://github.com/prometheus/blackbox_exporter
- Repositorio MySQL Exporter: https://github.com/prometheus/mysqld_exporter
- Dashboard Node Exporter Full (ID 1860): https://grafana.com/grafana/dashboards/1860
- Dashboard Blackbox Exporter (ID 7587): https://grafana.com/grafana/dashboards/7587
- Dashboard MySQL Overview (ID 7362): https://grafana.com/grafana/dashboards/7362
- Asistencia en la instalación y configuración: Claude (Anthropic)
