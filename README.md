
# Extremo a Extremo para el Despliegue Automatizado para Funciones de Red

A continuación se describe el proceso para hacer la instalación de servicios de red entre dos máquinas virtuales. Este proceso se llevará a cabo mediante el tenant de OpenStack, el cual constará de una bastión que centralizará el acceso mediante un túnel, y en la otra máquina estarán contenidos dos servicios: Prometheus y Grafana.

## Creación de máquinas virtuales para Bastion y Prometheus-Grafana

Para realizar el lanzamiento de las máquinas virtuales (VM), se debe acceder al tenant en el dashboard de OpenStack.

![Dashboard de OpenStack](https://docs.openstack.org/openstackdocstheme/latest/_images/dashboard-project-tab.png)

A continuación, se asignará un nombre y descripción (si es necesario). Para la bastión, se usará "uq-training-mateo-bastion". Luego se selecciona el servicio, en la pestaña de 'Origen', se escoge `Ubuntu Server 20.04 Focal`.

| Nombre | Actualizado   | Tamaño | Formato | Visibilidad |
|--------|---------------|--------|---------|-------------|
| auto-sync/ubuntu-focal-20.04-amd64-server-20221010-disk1.img | 10/11/22 6:27 AM | 575.19 MB | QCOW2 | Público |

En la sección 'Infraestructura' -> 'Sabor', como la VM solo va a ejercer la centralización del acceso, se selecciona el tamaño **m2.small** que tiene 2GB de RAM y 20GB de Disco.

| Nombre  | VCPUs | RAM  | Total de Disco | Disco Raíz | Disco Efímero | Público |
|---------|-------|------|----------------|------------|---------------|---------|
| m2.small| 1     | 2GB  | 20GB           | 10GB       | 10GB          | Sí      |

A continuación, se debe configurar una contraseña para la máquina y activar el protocolo SSH "siempre activo". En la pestaña 'Configuración', se debe pegar el siguiente bloque de código:

```bash
#cloud-config
password: <contraseña> # poner contraseña de preferencia
chpasswd: { expire: False } # que nunca expire la contraseña
ssh_pwauth: True # dejar siempre activa el protocolo
```

Para obtener el código completo, puede acceder al siguiente link [cloud-config.txt](https://osm.etsi.org/gitlab/vnf-onboarding/osm-packages/-/blob/master/slice_basic_vnf/cloud_init/cloud-config.txt)

> **NOTA**: El protocolo SSH nos permite realizar un acceso remoto a las VM. Para finalizar, se debe ejecutar la instalación.

Con esto, la VM de bastión estaría ejecutándose. Ahora se debe asignar una IP flotante para exponer la bastión al host. Vamos a "Computación" -> "Instancias", seleccionamos la VM, y en "Acciones", elegimos "Asignar IP flotante".

Una vez asignada la IP, se puede configurar el entorno de la bastión desde la terminal con el siguiente comando:

```bash
ssh ubuntu@10.0.100.110 # IP flotante de la bastión
```

Dentro de la bastión, se procederá a instalar el **Node Exporter**, utilizando los comandos del siguiente link [Node Exporter](https://prometheus.io/docs/guides/node-exporter/#installing-and-running-the-node-exporter):

```bash
wget https://github.com/prometheus/node_exporter/releases/download/v1.4.0/node_exporter-1.4.0.linux-amd64.tar.gz # Clonar repositorio de Node Exporter
tar xvfz node_exporter-1.4.0.linux-amd64.tar.gz # Descomprimir archivo .tar
cd node_exporter-1.4.0.linux-amd64 # Acceder a la carpeta
./node_exporter # Ejecutar el Node Exporter
```

Cuando se ejecuta el Node Exporter, comenzará a cargar las métricas. Para pausarlo, se presiona **Ctrl + Z**. Luego, se puede verificar que las métricas se están exportando con el siguiente comando:

```bash
curl http://localhost:9100/metrics # Verificar que las métricas se están exportando
```

Si no funciona el comando `curl`, primero se debe instalar la herramienta **net-tools**:

```bash
sudo apt install net-tools
sudo apt update
```

> **NOTA**: Si `curl` no funciona, instale las herramientas de red primero.

Por último, se le asignará un nombre a la bastión con las siguientes líneas de código:

```bash
sudo nano /etc/hosts # Asignar nombre al bastión
```

En el archivo que se abre, se agregará al lado de `localhost` el nombre de la bastión de la siguiente forma:

```bash
127.0.0.1 localhost 'uq-training-mateo-bastion' # Sin comillas simples
```

Para verificar que el Node Exporter está en funcionamiento, ejecute el siguiente comando:

```bash
ps aux | grep node_exporter
```

Con esto, la configuración de la bastión estará completa. Ahora, se debe crear la VM para **Prometheus-Grafana**. Se va a la dashboard de OpenStack -> Computación -> Instancias -> Lanzar instancia, y en la primera ventana 'Detalles', se le da el nombre "uq-training-mateo-prometheus-grafana".

En 'Origen', se selecciona el origen de arranque y se escoge **Ubuntu 20.04**. En 'Sabor', dado que la VM va a contener los dos servicios **Prometheus** y **Grafana**, se selecciona el tamaño **m2.large**:

| Nombre   | VCPUs | RAM  | Total de Disco | Disco Raíz | Disco Efímero | Público |
|----------|-------|------|----------------|------------|---------------|---------|
| m2.large | 4     | 8GB  | 50GB           | 40GB       | 10GB          | Sí      |

En la sección de configuración, se usa el mismo script del bastión. No es necesario que tengan la misma contraseña. Luego, se ejecuta la instancia.

Para acceder a Prometheus-Grafana, se hará mediante la bastión que centraliza el acceso:

```bash
ssh ubuntu@192.168.50.151 # IP de Prometheus-Grafana
```

Dentro de la VM de Prometheus-Grafana, el primer paso es instalar **Docker**. Para más información sobre Docker, se puede consultar este artículo [Docker](https://www.javiergarzas.com/2015/07/que-es-docker-sencillo.html).

La guía completa de instalación de Docker está disponible en [docker_installer](https://docs.docker.com/engine/install/ubuntu/).

```bash
sudo apt-get remove docker docker-engine docker.io containerd runc # Para eliminar cualquier registro de Docker en el sistema
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg lsb-release
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo   "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu   $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-compose-plugin
sudo apt update
```

Con Docker instalado, se debe asignar un nombre a la máquina, como se hizo con la bastión:

```bash
sudo nano /etc/hosts # Asignar nombre al Prometheus-Grafana
```

Se agregará el siguiente nombre:

```bash
127.0.0.1 localhost 'uq-training-mateo-prometheus-grafana'
```

> **NOTA**: Este paso se realiza para optimizar las funciones de la VM.

A continuación, se le darán permisos de superusuario al comando Docker para evitar escribir `sudo` en cada comando:

```bash
sudo usermod -aG docker $USER
```

Para que el cambio surta efecto, se debe salir de la VM con `exit` y luego volver a ingresar. Para verificar que Docker está funcionando, se puede ejecutar:

```bash
docker ps -a
```

Luego, se crea un archivo llamado `prometheus.yml`, que contendrá la configuración del servidor Prometheus:

```bash
sudo mkdir -p /etc/prometheus
sudo touch /etc/prometheus/prometheus.yml
sudo nano /etc/prometheus/prometheus.yml
```

El archivo debe contener lo siguiente:

```bash
global:
  scrape_interval: 15s

scrape_configs:
- job_name: node
  static_configs:
  - targets: ['192.168.50.238:9100'] # IP de la bastión (no flotante)
```

Ahora, se ejecuta Prometheus con el siguiente comando:

```bash
docker run -d --name=prometheus -p 9090:9090 -v /etc/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml prom/prometheus
```

Para verificar que Prometheus está funcionando, ejecute:

```bash
docker ps -a
```

Para verificar los targets, se puede hacer un túnel SSH de la siguiente manera:

```bash
ssh ubuntu@10.0.100.110 -L 9091:192.168.50.151:9090
```

> **NOTA**: Para más información sobre túneles SSH, consulte [SSH_TUNEL](https://www.concordia.ca/ginacody/aits/support/faq/ssh-tunnel.html#:~:text=SSH%20tunneling%2C%20or%20SSH%20port,machine%20via%20a%20secure%20channel).

Luego, en el navegador, acceda a `http://localhost:9091`.

Verifique que el Target que representa la bastión esté "UP".

Finalmente, para instalar **Grafana**, ejecute el contenedor con el siguiente comando:

```bash
docker run -d --name=grafana -p 3000:3000 grafana/grafana # Ejecutar en la VM de Prometheus-Grafana
```

Verifique que el contenedor esté funcionando con:

```bash
docker ps -a
```

Para acceder a Grafana, se debe hacer un túnel SSH y luego ingresar en el navegador `http://localhost:3001`.

![Grafana](https://grafana.com/static/img/grafana_labs.jpg)
![Grafana 2](https://miro.medium.com/max/3694/1*KimwgjULRZzONpjGFH1sTA.png)

> **NOTA**: Antes de acceder a Grafana, haga un túnel como en el caso de Prometheus con el siguiente comando:

```bash
ssh ubuntu@10.0.100.110 -L 3001:192.168.50.151:3000
```

En la interfaz de Grafana, acceda a "Configuration" -> "Data Sources" -> "HTTP" y en "URL" coloque la IP de **Prometheus-Grafana** con el puerto 9090: `http://192.168.50.151:9090` y haga clic en "Save & Test". Luego, acceda a "Dashboards" -> "Import" e ingrese el ID 1860 para cargar el dashboard.
