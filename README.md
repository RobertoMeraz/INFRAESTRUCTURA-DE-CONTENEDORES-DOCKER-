# INFRAESTRUCTURA-DE-CONTENEDORES-DOCKER-
# Arquitectura e Infraestructura: Clúster Docker Swarm Seguro

## 1. Resumen y Contexto Arquitectónico
Esta infraestructura distribuye servicios y contenedores sobre un entorno de virtualización Proxmox, utilizando Docker Swarm para la orquestación. El diseño resuelve los problemas de sobre-aprovisionamiento de máquinas virtuales e integra una capa de *Security by Design*. Esto incluye un WAF (Web Application Firewall) para mitigar ataques de capa 7 y un SIEM (Wazuh) para la detección de intrusiones.

**Especificaciones de Nodos:**
* **SO:** Debian 13 (Trixie) - Kernel 6.12.86+deb13-amd64.
* **Motor:** Docker 29.4.3.
* **Topología Base:** 1 Manager / Leader y múltiples Workers.

## 2. Topología de Red y Segmentación
La red está estrictamente segmentada usando redes overlay de Swarm y bridge locales para cumplir con las políticas de seguridad.
* **`red_mix` (10.2.0.0/23):** Red principal de servicios y monitoreo.
* **`red_web` (10.1.0.0/23):** Red secundaria para aislamiento de aplicaciones expuestas a internet.
* **`single-node_default` (172.19.0.0/16):** Red aislada (Bridge) exclusiva para la gestión del SIEM (Wazuh).

### Flujo de Tráfico y WAF (Nginx Proxy Manager + ModSecurity)
Todo el tráfico externo pasa por un flujo de inspección obligatorio antes de tocar los contenedores backend:
1. **Ingreso:** El usuario accede vía HTTPS desde Internet.
2. **NPM (`npm_app`):** Recibe en puertos 80/443 y gestiona los certificados SSL.
3. **WAF (`waf_waf-central`):** NPM redirige todo al puerto del host bridge. ModSecurity inspecciona la petición utilizando reglas OWASP.
4. **Backend:** Si el tráfico es limpio, el WAF enruta la petición al servicio final (ej. `web-hello`, `app-python`) basándose en el encabezado Host.

## 3. Acceso Seguro mediante Túnel SSH (Port Forwarding)
Por motivos de estricta seguridad, las interfaces de administración, bases de datos y dashboards de monitoreo no están expuestas al tráfico web público ni a la WAN. Para interactuar con los paneles de gestión, es obligatorio establecer un canal cifrado a través de un túnel SSH (Local Port Forwarding).

```bash
ssh -L 9443:<IP_MANAGER>:9443 -L 81:<IP_MANAGER>:81 -L 3000:<IP_MANAGER>:3000 -L 9090:<IP_MANAGER>:9090 -L 8443:<IP_MANAGER>:8443 <USER>@<IP_MANAGER>
```
*(Nota: Reemplazar `<IP_MANAGER>` y `<USER>` con las credenciales de administración autorizadas).*

**Mapeo de Puertos Locales Redirigidos:**
Una vez establecida la sesión SSH, los servicios quedan mapeados localmente:
* **Portainer HTTPS UI:** `https://localhost:9443` — Gestión gráfica del clúster.
* **Nginx Proxy Manager UI:** `http://localhost:81` — Administración del proxy inverso y certificados.
* **Grafana Dashboards:** `http://localhost:3000` — Visualización de analíticas y métricas.
* **Prometheus Web UI:** `http://localhost:9090` — Consola de estado de targets y consultas PromQL.
* **Wazuh Dashboard:** `https://localhost:8443` — Consola central de seguridad (SIEM).

## 4. Stack de Monitoreo (Prometheus + Grafana)
Debido a las restricciones de firewall (Hairpin NAT) y la malla de enrutamiento de Swarm, la recolección de métricas utiliza una solución de *Service Discovery* automatizada. Se utiliza un script (vía Cron, ejecutado cada 5 minutos) que consulta al orquestador las IPs internas de `node-exporter` y `cadvisor`, generando archivos JSON. Prometheus lee estos archivos dinámicamente, asegurando que las métricas no se mezclen por el balanceador de carga.

**Volúmenes Críticos (Persistencia):**
* `monitoreo_prometheus_data`: `/prometheus` (Datos TSDB).
* `monitoreo_grafana_data`: `/var/lib/grafana` (Base de datos SQLite de dashboards).
* `wazuh-indexer-data`: `/var/lib/wazuh-indexer` (Índices SIEM).

## 5. Mantenimiento y Troubleshooting Operativo

### Actualización y Recarga de Servicios
Para forzar la recarga de un contenedor tras cambiar configuraciones:
```bash
docker service update --force <nombre_servicio>
```
Para actualizar la imagen a la última versión:
```bash
docker service update --image <nueva_imagen> <nombre_servicio>
```

### Recuperación de Grafana (DB SQLite)
En caso de corrupción de la base de datos de Grafana (loop de reinicios):
1. Escalar el servicio a 0 réplicas:
   `docker service scale monitoreo_grafana=0`
2. Eliminar la DB corrupta montando el volumen en un contenedor temporal:
   `docker run --rm -v monitoreo_grafana_data:/data alpine rm -f /data/grafana.db`
3. Escalar de nuevo a 1 réplica:
   `docker service scale monitoreo_grafana=1`
*(Los Datasources se re-aprovisionarán automáticamente desde los archivos de configuración estáticos)*.

### Administración del Clúster Swarm
Para agregar un nodo Worker al clúster, ejecutar en el Manager:
```bash
docker swarm join-token worker
```
Para retirar un nodo de la malla, ejecutar directamente en la máquina a eliminar:
```bash
docker swarm leave
```
