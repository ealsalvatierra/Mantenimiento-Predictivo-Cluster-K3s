Clúster Hibrido K3s: Mantenimiento Predictivo Industrial
---
📐 Arquitectura del Clúster
El entorno está distribuido en nodos con roles diferenciados para garantizar la estabilidad del plano de control y un rendimiento óptimo en el procesamiento de cargas:
Nodo	Rol	Recursos Recomendados	Componentes / Cargas de Trabajo
`master-node`	Control Plane	2 vCPUs (1x2), 4 GB RAM	K3s API Server, SQLite (`kine`), CoreDNS, Traefik
`worker1-node`	Worker Node	2 vCPUs (1x2), 2–4 GB RAM	Pods del namespace `predictivo` (Mosquitto, InfluxDB, Grafana, Classifier)
`worker2-pi`	Worker Node (Edge)	2–4 Cores, 2–4 GB RAM	Captura de sensores en el borde / Emisor
---
📁 Estructura del Repositorio
```text
.
├── README.md                           # Documentación principal del proyecto
├── manifests/                          # Manifiestos YAML de Kubernetes
│   ├── 00-namespace.yaml               # Creación del namespace 'predictivo'
│   ├── 01-mosquitto-deployment.yaml   # Broker MQTT
│   ├── 02-influxdb-deployment.yaml    # Base de datos de series temporales
│   ├── 03-grafana-deployment.yaml     # Tableros de monitoreo y visualización
│   └── 04-classifier-deployment.yaml  # Clasificador / Procesador de sensores
└── docs/
    ├── comandos-despliegue.txt        # Guía de instalación y comandos rápidos
    └── solucion-de-problemas.txt      # Troubleshooting y recuperación ante fallos
```
---
🚀 Guía de Despliegue Rápido
1. Preparación de Hardware (Hipervisor / VMs)
Asegúrate de asignar la siguiente topología en tu hipervisor (VirtualBox, Hyper-V o VMware):
Master Node: 1 Socket, 2 Núcleos (2 vCPUs) y mínimo 4 GB de RAM.
Worker Node: 1 Socket, 2 Núcleos (2 vCPUs) y 2 a 4 GB de RAM.
2. Despliegue de los Manifiestos
Desde el nodo maestro (`master-node`), ejecuta en orden:
```bash
# 1. Crear el namespace
kubectl apply -f manifests/00-namespace.yaml

# 2. Desplegar los servicios de la aplicación
kubectl apply -f manifests/

# 3. Verificar que los Pods estén en estado Running
kubectl get pods -n predictivo -o wide
```
---
🛠️ Guía de Diagnóstico y Mantenimiento
Verificación de Recursos del Nodo
Si el API Server no responde o experimentas retrasos en la respuesta, verifica la carga de procesador y memoria:
```bash
# Diagnóstico rápido de CPU, Memoria, Disco y errores OOM
free -h
top -b -n 1 | head -n 12
df -h / /var/lib/rancher/k3s
sudo dmesg -T | grep -i oom
```
Solución a Saturación de CPU / Error `TLS Handshake Timeout`
En caso de reinicios abruptos del anfitrión o bloqueos por saturación de CPU (`0.0% id`), ejecuta esta secuencia de limpieza e inicialización en el `master-node`:
```bash
sudo systemctl stop k3s
sudo pkill -9 -f runc
sudo pkill -9 -f containerd
sudo pkill -9 -f k3s
sudo systemctl restart systemd-timesyncd
sudo systemctl start k3s
sleep 20
sudo kubectl get nodes -o wide
```
Inspección de Logs por Componente
Para consultar los registros de un servicio específico dentro del namespace `predictivo`:
```bash
# Ver Pods y sus etiquetas asignadas
kubectl get pods -n predictivo --show-labels

# Consultar logs mediante etiqueta app
kubectl logs -n predictivo -l app=influxdb -f

# Consultar logs directamente por nombre de Pod
kubectl logs -n predictivo <NOMBRE_DEL_POD> -f
```
---
📊 Visualización en Grafana ("No Data")
Si tras un reinicio del clúster el panel de Grafana muestra "No Data":
Confirma que el Pod de InfluxDB esté en estado `Running`: `kubectl get pods -n predictivo`.
Asegúrate de que el script emisor o sensor esté publicando datos en Mosquitto.
En la interfaz web de Grafana, cambia el rango de tiempo superior derecho a `Last 15 minutes` para visualizar la ventana de tiempo activa tras el reinicio.
