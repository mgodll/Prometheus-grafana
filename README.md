# Extremo a Extremo para el Despliegue Automatizado para Funciones de Red

Acontinuación se describira el proceso para hacer la instalacion de servicios de red entre dos maquinas virtuales, se realizara mediante el tenant de Openstack, el cual constara de una bastion que centralizara el acceso mediante un tunel,en la otra maquina estara contenido dos servicios que son prometheus y grafana

## Creación de maquinas virtuales para bastion y prometheus-grafana

Para realizar el lanzamiento de las maquinas virtuales (VM), se procede acceder al tenant de la dashboard en openstack 

![](https://docs.openstack.org/openstackdocstheme/latest/_images/dashboard-project-tab.png)

Para lo cual se le dara un nombre y descripcion de ser necesario, en el caso de la bation "uq-training-mateo-bastion", despues se selecciona el servicio, en la pestaña de 'Origen' `ubunut server 20.04 focal`

| Nombre | Actualizado   | Tamaño | Formato | Visibilidad |
|---------|--------------|------|-----|----|
|auto-sync/ubuntu-focal-20.04-amd64-server-20221010-disk1.img| 10/11/22 6:27 AM | 575.19 MB | QCOW2 | Público |

 Paso siguiente la infraestructura -> 'Sabor*', como la VM solo va a ejercer la centralizacion del acceso se le pondra m2.small, que tiene 2GB de RAM, 20 GB de Disco

| Nombre | VCPUS | RAM | Total de Disco | Disco Raíz | Disco efimero | Público |
|---------|--------------|------|-----|----|--|--|
|m2.small| 1 | 2 GB | 20GB | 10GB | 10GB | Si |

Como ultimo se realizara la configuracion para darle una contaseña a la maquina y para activar el protocolo ssh "siempre activo", en la pestaña 'configuración', se pegara las siguientes linas de codigo

```bash
#cloud-config
password: <contraseña> # poner contraseña de preferencia
chpasswd: { expire: False } # que nunca expire la contraseña
ssh_pwauth: True # dejar siempre activa el protocolo
```
Para obtener el codigo completo puede acceder al siguiente link [cloud-config.txt](https://osm.etsi.org/gitlab/vnf-onboarding/osm-packages/-/blob/master/slice_basic_vnf/cloud_init/cloud-config.txt)

> **NOTA** Es de resaltar que el protocolo ssh nos permite ejercer un acceso remoto a las VM, para finalizar le damos ejecutar instalación.

Con esto se tendria la VM de bastion ejecutandoce, ahora tendremos que asignar una ip flotante, con el proposito de darle a mostrar todo lo que esta despues de la bastion al host, nos vamos a computacion -> instancias, aqui seleccionamos la VM, vamos al apartado 'actions', en esta se selecciona asignar ip flotante.

Ya con la ip asignada solo nos queda configurar todo el entorno de la bastion, para ello abrimos la terminal de preferencia y escribimos lo siguiente

```bash

ssh ubuntu@10.0.100.110 #Ip flotante del bastion

```
Dentro del bastion se procedera a instalar el node exporter, en el siguiente link se encontrara toda la información [node-exporter](https://prometheus.io/docs/guides/node-exporter/#installing-and-running-the-node-exporter)

```bash

wget https://github.com/prometheus/node_exporter/releases/download/v1.4.0/node_exporter-1.4.0.linux-amd64.tar.gz # clonar repositorio de node exporter

tar xvfz node_exporter-1.4.0.linux-amd64.tar.gz # descomprimir el archivo .tar

cd node_exporter-1.4.0.linux-amd64 # acceder a la carpeta

./node_exporter # ejecutar el node exporter
```
Cuando se ejecuta node exporter comenzara a cargar las metricas, para pausar esta acción le daremos 'control + z', despues se ingresara el siguiente comando

```bash

curl http://localhost:9100/metrics # verificar que las métricas se están exportando

```

Paso siguiente instalar el 'net-tools', que es una herramienta de red con la cual nos permite usar comandos como ifocnfig

```bash

sudo apt install net-tools
sudo apt update

```

> **NOTA** En caso de no funcionar el curl primero se instala las herramientas de red

Por ultimo se le asignara un nombre a la bastion con las siguientes lineas de codigo

```bash

sudo nano /etc/hosts # para asignar nombre del bastion

```

En el escript que se abre se le asignara al lado del localhost el nombre que se le dio a la bastion de la siguiente forma

```bash

127.0.0.1 localhost 'uq-training-mateo-bastion' #sin comillas simples

```
Para verificar que el node exporte esta en funcionamiento se ejecuta la siguiente linea de codigo

```bash

ps aux | grep node_exporter

```

Con lo anterior la configuración de la bastion estaria completa, se procede a crear la VM para prometheus-grafana, nos vamos a la dashboard de openstack -> computacion -> instancias -> lanzar intancias, la primer venta 'Detalles' le pondres nombre como en el caso de la bastion, para este trabajo se le da el siguiente 'uq-training-mateo-prometheus-grafana'.

Ahora en 'origen' en seleccionar origen de arranque se escoge Si, en tamaño de volumen Si, el servicio de linux la misma opcion de la bastion Ubuntu 20.04; en la siguiente interfaz de 'sabor*' como la VM va a contener los dos servicios, `prometheus y grafana` se le da una infraestructura con mejores recursos para ello se escoge m2.large:

| Nombre | VCPUS | RAM | Total de Disco | Disco Raíz | Disco efimero | Público |
|---------|--------------|------|-----|----|--|--|
|m2.large| 4 | 8 GB | 50GB | 40GB | 10GB | Si |

Por ultimo en la interfaz configuración, se pondra el mismo script del bastion, no es necesario que tengan la misma contraseña, y cerramos con ejecutar instancia

Para acceder a prometheus-grafana se realizara mediante la interfaz de bastion ya que esta es la que se encarga de centralizar el acceso

```bash

ssh ubuntu@192.168.50.151 #Ip de prometheus-grafana

```
Dentro de la MV de prometheus-grafana, el primer paso para configurar los servicios es instalar docker, "es un proyecto de código abierto que automatiza el despliegue de aplicaciones dentro de contenedores de software, proporcionando una capa adicional de abstracción y automatización de virtualización de aplicaciones en múltiples sistemas operativos", para mas informacion sobre docker acceder al siguiente vinculo [docker](https://www.javiergarzas.com/2015/07/que-es-docker-sencillo.html)

La guia completa para instalar docker la encontraremos en [docker_installer](https://docs.docker.com/engine/install/ubuntu/)

![](https://w7.pngwing.com/pngs/219/411/png-transparent-docker-logo-kubernetes-microservices-cloud-computing-dockers-logo-text-logo-cloud-computing.png)

```bash

sudo apt-get remove docker docker-engine docker.io containerd runc #para eliminar cualquier registro de docker en el sistema

sudo apt-get update

sudo apt-get install     ca-certificates     curl     gnupg     lsb-release

sudo mkdir -p /etc/apt/keyrings

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update

sudo apt-get install docker-ce docker-ce-cli containerd.io 

docker-compose-plugin

sudo apt update
```

Con docker instalado nos queda dar un nombre a la maquina como en la bastion

```bash

sudo nano /etc/hosts # para asignar nombre del prometheus-grafana

```
Nombre
```bash

127.0.0.1 localhost 'uq-training-mateo-prometheus-grafan'

```

>**NOTA** Este paso se realiza con el fin de optimizar las funciones de las VM

Despues le daremos permisos de super usario al comando docker, con el fin de ahorrar el digitar sudo docker

```bash

sudo usermod -aG docker $USER

```
Para que el cambio surta efecto nos saldremos de la VM con `exit` y volveremos a entrar, para verificar que si esta funcionando escribimos en la terminal `docker ps -a`

Ahora vamos a crear un archivo que lo llamaremos `prometheus.yml` que va a contener la configuración del servidor de Prometheus

```bash

sudo mkdir -p /etc/prometheus
sudo touch /etc/prometheus/prometheus.yml
sudo nano /etc/prometheus/prometheus.yml

```
El archivo debe de contener lo siguiente

```bash

global:
  scrape_interval: 15s

scrape_configs:
- job_name: node
  static_configs:
  - targets: ['192.168.50.238:9100'] # ip del bastion pero no la flotante

```
Tendra la ip del dominio bastion y el puerto 9100 porque es el puerto donde se está ejecutando el servicio del agente que se instalo en el punto anterior

Ahora ejecutamos el prometheus, y su servicio con el siguiente comando

```bash

docker run -d --name=prometheus -p 9090:9090 -v /etc/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml prom/prometheus

```
se valida el contenedor usando el comando `docker ps -a` en la maquina virtual de docker, prometheus-grafana, ahora verificaremos el target haciendo un tunel de la siguiente manera

```bash

ssh ubuntu@10.0.100.110 -L 9091:192.168.50.151:9090

```
> **NOTA:** Para más información de túneles SSH pueden consultar [SSH_TUNEL](https://www.concordia.ca/ginacody/aits/support/faq/ssh-tunnel.html#:~:text=SSH%20tunneling%2C%20or%20SSH%20port,machine%20via%20a%20secure%20channel), es de resaltar que el tunel se realiza en la interfaz del host no en las maquinas virtuales.

Ahora solo queda ingresar al navegador de la maquina virtual http://localhost:9091

Vamos hacer click en Status -> Targets y se verificara que el Target que representa la Bastion se encuentre UP.

Por ultimo se instalara el grafana, para hacer esto ejecutaremos el contenedor con el servicio usando

```bash

docker run -d --name=grafana -p 3000:3000 grafana/grafana # se realiza en la VM de prometheus-grafana

```

verificaremos con el comando `docker ps -a`

Para acceder al grafana ponemos en la web http://localhost:3001

![](https://grafana.com/static/img/grafana_labs.jpg)
![](https://miro.medium.com/max/3694/1*KimwgjULRZzONpjGFH1sTA.png)

> **NOTA:** Antes de acceder al grafana tendremos que hacer un tunel como en el caso de prometheus con el siguiente comando `ssh ubuntu@10.0.100.110 -L 3001:192.168.50.151:3000` donde va la ip flotante del bastion y la ip del prometheus-grafana el cambio va en el puerto

En la interfaz de grafana accederemos a configuration -> data sources -> Http y en url se pondra la ip del prometheus-grafana con su puerto 9090 `http://192.168.50.151:9090` y le daremos `save & test`, despues le daremos en dashboards -> import y en Import via grafana.com pondremos 1860 y despues load
