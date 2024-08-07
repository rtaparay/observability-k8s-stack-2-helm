# Observability and logging in AWS EKS
## Stack: Prometheus, Alertmanager, Grafana, Loki y Fluent Bit

## Lista de herramientas y controladores de Kubernetes

- **EBS CSI Driver** (EKS Addon)
- **Prometheus Operator** (usando el helm chart kube-prometheus-stack)
- **Alertmanager** (usando el helm chart kube-prometheus-stack)
- **Grafana** (usando el helm chart kube-prometheus-stack)
- **Loki** (usando el helm chart de Grafana)
- **Fluent-Bit** (usando el helm chart de Fluent-Bit)

## Historia

1. Instalación de Prometheus Operator y Grafana en el clúster EKS.
2. Configuración de reglas de alerta, monitores de servicio y AlertManager para alertas por correo electrónico.
3. Instalación de Loki en el clúster EKS y configuración con AWS S3 para el almacenamiento de registros.
4. Instalación de Promtail en el clúster EKS y configuración para enviar registros a Loki.
5. Configuración de Grafana para mostrar los registros de la aplicación.

## Lista de servicios de AWS

- **Amazon EKS**
- **Amazon EC2**

## ☸️ Monitoreo

El monitoreo implica rastrear el rendimiento de su aplicación y sus recursos, y enviar alertas cuando algo funciona lentamente o falla, para evitar que los problemas se agraven.

### 📊 Prometheus

Prometheus es una herramienta de monitoreo de código abierto que rastrea su carga de trabajo y almacena todas sus métricas en una base de datos de series de tiempo. Usamos PromQL para consultar las métricas. En este blog, almacenaremos datos dentro de un volumen de AWS EBS.

### 📢 Alertmanager

Alertmanager es un componente de Prometheus responsable de enviar alertas a los usuarios.

### 📜 Fluent-Bit

Fluent Bit es un procesador de logs rápido y flexible, compatible con varios sistemas operativos. Se utiliza para enrutar los logs a varios destinos de AWS, como Amazon CloudWatch, Amazon Kinesis Data Firehose, Amazon Simple Storage Service (Amazon S3) y Amazon OpenSearch. Para esta POC, recopila todos los registros de los contenedores y los envía a Loki.

### 🔗 Loki

También es una herramienta de código abierto diseñada y desarrollada por Grafana Labs. Consume datos enviados por Promtail u otras herramientas, los procesa y los filtra. Usamos LogQL para consultar los registros de Loki. Loki se puede integrar con muchos servicios en la nube; en este blog usaremos el bucket AWS S3 para almacenar los registros.

###  🖥️Grafana:

Es una herramienta de visualización comúnmente utilizada para monitoreo y registro.
Grafana se puede integrar con Prometheus, Loki y muchas otras herramientas para crear un hermoso panel de control. Grafana consultará a Prometheus y Loki para obtener las métricas y los registros.

🎯 Architecture:

![image](./arquitectura.png)  

Como puede ver en la arquitectura, Prometheus extrae métricas de la aplicación y el clúster y las almacena en volúmenes AWS EBS para mantenerlas persistentes en caso de falla del pod. De la misma manera, Grafana y Alermanger también almacenarán sus datos dentro del volumen EBS.

Promtail recopilará todos los registros de los nodos (registros de aplicaciones + registros de componentes) y enviará esos registros a Loki.

Loki agregará y procesará los registros y los enviará al depósito AWS S3.

Grafana consultará a Prometheus y Loki para obtener métricas y registros.


## 🚀 Guía paso a paso:
## Instalar el repo necesarios:

helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

helm repo add grafana https://grafana.github.io/helm-charts

helm repo update

## Instalar y configurar Grafana + Prometheus + Alertmanager:

Ahora, instalemos el operador Prometheus en el clúster AWS EKS usando el gráfico Helm.

kubectl create ns observability

helm search repo kube-prometheus-stack --versions | grep v0.75.0

helm show values prometheus-community/kube-prometheus-stack --version=61.3.0 > observability_stack_values.yaml

helm install observability-stack prometheus-community/kube-prometheus-stack --version=61.3.0 -n observability -f prometheus/custom_observability_stack_values.yaml

kubectl -n observability get service
kubectl -n observability get pods -l "release=observability-stack"

Es hora de aplicar todas estas configuraciones. Ejecute el siguiente comando:

kubectl apply -k observability-k8s-stack-2-helm/

kubectl apply -f kustomization.yml

Necesitamos esperar un par de minutos para que el operador de Prometheus recargue su configuración.

## Visite la interfaz de Grafana 

kubectl port-forward -n observability service/observability-stack-grafana 8080:80
http://localhost:8080

Verá muchos paneles de control predefinidos. Puede utilizarlos para realizar un seguimiento o diseñar o importar los suyos propios.

Haga clic en el New botón de la parte superior derecha, seleccione Importen el menú desplegable e importe el panel.

Importe el panel de la comunidad escribiendo 315 y seleccionando Loki como fuente de datos.

### Guia de alertas:
- https://samber.github.io/awesome-prometheus-alerts/rules#kubernetes

Visualizar contraseña de admin:

kubectl -n observability get secret observability-stack-grafana -o jsonpath="{.data.admin-password}" | base64 --decode ;echo

kubectl -n observability get secret observability-stack-grafana -o jsonpath="{.data.admin-user}" | base64 --decode ;echo

### Cargar Datasources y dashboards:

kubectl create secret generic datasources-secret -n observability --from-file=grafana/datasources/datasources.yaml

kubectl delete secret datasources-secret -n observability

kubectl label secret datasources-secret -n observability grafana_datasource=1

cd grafana/dashboards

ls -1 *.json | sed 's/\.[^.]*$//' | xargs -I {arg} kubectl create configmap -n observability --from-file={arg}.json {arg}-dashboard-cm

kubectl get configmaps -n observability | grep dashboard-cm | awk '{print $1}' | xargs -I {arg} kubectl label cm {arg} -n observability grafana_dashboard=1

kubectl get cm,secret -n observability --show-labels | grep -E "NAME|dashboard-cm|datasource"

## Visite la interfaz de usuario de Prometheus:

kubectl port-forward -n observability service/prometheus-operated 9090:9090
- http://localhost:9090

Para consultar las reglas aplicadas, haga clic en el Alertsbotón de la parte superior.

Es hora de configurar alertas personalizadas, un Alertmanager para recibir correos electrónicos y un ServiceMonitor para recopilar las métricas de nuestra aplicación.

Antes de configurar Alertmanager, necesitamos credenciales para enviar correos electrónicos estoy usando Gmail, pero se puede usar cualquier proveedor SMTP como AWS SES. Así que obtengamos las credenciales para eso.
Abra la configuración de su cuenta de Google y busque App password y cree una nueva contraseña.

Convierte esa contraseña al formato base64.

Generar token de aplicación: 
- https://support.google.com/accounts/answer/185833?hl=es&sjid=17517275735111420684-SA
- https://www.base64encode.org/

datasource: 
    - http://prometheus-server.observability.svc.cluster.local


## Visite la interfaz de usuario de Alertmanager:

kubectl port-forward -n observability service/alertmanager-operated 9093:9093
- http://localhost:9093

Haga clic en el Status botón de la parte superior para ver las configuraciones aplicadas.

Ahora, bloqueemos la aplicación Node.js dos veces para recibir alertas de Alertmanager.
La aplicación Nodejs tiene una ruta /crash que bloquea el contenedor y Kubernetes lo reinicia automáticamente. Sin embargo, si la aplicación se bloquea más de 2 veces, Alertmanager enviará una alerta a nuestro correo electrónico.


### Glosario de los campos de alerta:

alert:  El nombre de la alerta ( KubernetesPodNotHealthy).

expr: expresión de Prometheus que se evaluará. Esta alerta se activa si se detecta algún pod en un estado: Pending, Unknown, Failed

for: la duración durante la cual la condición debe ser verdadera antes de que se active la alerta (1m or 1 minute).

labels: etiquetas adicionales para categorizar la alerta. En este caso, la etiquetamos con una gravedad de critical.

annotations: información descriptiva sobre la alerta. Estos campos pueden brindar contexto cuando se activa la alerta

— — summary: Una breve descripción de la alerta (Kubernetes Pod not healthy (instance {{ $labels.instance }})).

— — description: Una descripción detallada que incluye valores dinámicos de las etiquetas de alerta: (Pod {{ $labels.namespace }}/{{ $labels.pod }} has been in a non-running state for longer than 15 minutes.\n VALUE = {{ $value }}\n LABELS = {{ $labels }}).

## Visite la interfaz de usuario de Prometheus

Compruebe la alerta en el estado de activación ejecutando, Verifique que Alertmanager recibió una alerta de Prometheus:

kubectl port-forward -n observability service/prometheus-operated 9090:9090

Nota: 
- Debería recibir una notificación por correo electrónico en su dirección de correo configurada.
- Lo configuramos para enviar correos electrónicos cada 5 minutos hasta que se resuelva el problema.

## Instalación de Loki

Hemos configurado la monitorización, ahora configuremos Loki.
Ya agregamos el repositorio de Grafana Helm en el paso anterior, que incluye tanto a Loki como fluent-bit.

Ahora estamos listos para configurar Loki:

helm search repo grafana/loki-distributed --versions | grep 2.9.6

helm show values grafana/loki-distributed --version=0.79.0 > loki/loki_distributed_values.yaml

- Nota: custom_loki_distributed_values.yml Tiene todas las configuraciones predeterminadas, pero tenemos que hacer algunos cambios para configurar el bucket AWS S3.
- Asegúrese de agregar el nombre de su depósito, la región, el ID de acceso y el ID de acceso secreto.

helm install loki grafana/loki-distributed --version=0.79.0 --namespace=observability 

Ahora, importe el panel de la comunidad escribiendo 15141 y seleccionando Loki como fuente de datos.

Ahora, sigamos adelante y veamos nuestros registros en el panel de Grafana.

Antes de agregar un nuevo panel, necesitamos agregar nuevas fuentes de datos para que Grafana pueda consultar registros de Loki.

datasource: http://loki-loki-distributed-gateway.observability.svc.cluster.local

## Instalación de Fluent-Bit

Ahora, configuremos el recopilador de registros, Fluent-Bit. 

helm repo add fluent https://fluent.github.io/helm-charts

helm repo update

helm search repo fluent/fluent-bit --versions | grep 2.0.8

helm show values fluent/fluent-bit --version=0.22.0 > fluent_bit_values.yml

Instalamos fluent-bit con las configuraciones personalizadas:

helm install fluent-bit fluent/fluent-bit --version=0.22.0 --namespace=observability -f fluent-bit/custom_fluent_bit_values.yaml


Documentacion oficial: https://docs.fluentbit.io/manual/v/2.0/pipeline/outputs/loki

Ahora, intentemos generar registros desde su aplicación seleccionando el default namespace o el namespace que tiene creado en el menú desplegable en la parte superior.

## Limpieza

kubectl delete -k app/

helm uninstall observability-stack -n observability

helm uninstall loki -n observability

helm uninstall fluent-bit -n observability

kubectl delete ns observability

kubectl get pv -n observability # eliminar en caso que exista