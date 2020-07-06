# Dashboard de actividad volcánica con ELK

Este proyecto consiste en la implementación de la pila ELK para la visualización de datos sobre actividad volcánica significativa en el mundo.

Cada una de nuestras aplicaciones se encontrará corriendo en un contenedor de Docker y se comunicarán entre ellas para integrar nuestro sistema.

La información fue extraída del repositorio de datos del Departamento de Seguridad Nacional de los Estados Unidos ([HIFLD](https://hifld-geoplatform.opendata.arcgis.com/datasets/3ed5925b69db4374aec43a054b444214_6/data)).

![Dashboard de actividad volcánica](./dashboard_volcanoes.png)

## Prerrequisitos

El proyecto está pensado para trabajar con tres máquinas virtuales conectadas a una red interna en común. Nos referiremos a estas máquinas como **vm1**, **vm2** y **vm3**. Los requerimientos de memoria y almacenamiento mínimos recomendados son los siguientes:

|Máquina|RAM    |Almacenamiento|
|-------|------:|-------------:|
|**vm1**|2 GB   |16 GB         |
|**vm2**|1.5 GB |10 GB         |
|**vm3**|1.5 GB |10 GB         |

Para levantar el sistema es necesario tener instalado Docker y Docker Compose en cada una de estas máquinas. A continuación, se dan las instrucciones para su instalación en Ubuntu Server 18.04, que fue el sistema operativo utilizado para el desarrollo, para otras instalaciones consultar la documentación de [Docker](https://docs.docker.com/get-docker/) y [Docker Compose](https://docs.docker.com/compose/install/).

**Nota:** Durante el desarrollo de este proyecto se usó **VirtualBox** como software de virtualización, la utilización de un software de virtualización diferente puede alterar la forma en la que se realizan algunas configuraciones.

### Instalación de Docker

Actualizar los repositorios de software:
```
sudo apt-get update
```
Descargar e instalar Docker:
```
sudo apt install docker.io
```
Iniciar Docker:
```
sudo systemctl start docker
```
Habilitar el servicio de Docker para iniciar automáticamente en el arranque:
```
sudo systemctl enable docker
```
Comprobar la versión instalada:
```
sudo docker --version
```

### Instalación de Docker Compose

Descargar Docker Compose (**Nota:** aquí se está descargando la versión 1.26.1, para descargar la versión más reciente o alguna otra consultar la [documentación](https://docs.docker.com/compose/install/)).
```
sudo curl -L "https://github.com/docker/compose/releases/download/1.26.1/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
```
Asignar permisos de ejecución al binario:
```
sudo chmod +x /usr/local/bin/docker-compose
```
Comprobar la versión instalada:
```
sudo docker-compose --version
```

## Configuración

A continuación, se describe la preparación de la infraestructura para correr el sistema. Esta configuración debe realizarse en cada una de las máquinas virtuales (vm1, vm2 y vm3).

### Configuración de las máquinas virtuales

Estos son los pasos a seguir para que nuestro sistema Ubuntu corra sin problemas la pila de ELK, esta configuración puede variar en otros sistemas operativos.

#### Configuración del sistema

Es necesario incrementar el parámetro `max_map_count` del kernel de linux para evitar quedarse sin áreas de mapeo. Para ello debemos agregar la siguiente línea al archivo `/etc/sysctl.conf`:
```
vm.max_map_count=262144
```
Recargamos la configuración:
```
sysctl -p
```
Comprobamos el nuevo valor:
```
cat /proc/sys/vm/max_map_count
```

Debemos configurar también los límites de memoria para nuestro usuario. Para ello editamos el archivo `/etc/security/limits.conf` y añadimos las siguientes líneas (**Nota:** el usuario que tenemos asignado para la ejecución del sistema en todas las máquinas es el usuario **elastic**, en caso de tener un nombre de usuario diferente, sustituir el nombre de elastic):
```
elastic soft memlock unlimited
elastic hard memlock unlimited
```

Para aplicar los cambios debemos volver a iniciar sesión.

#### Configuración de red

Para facilitar la comunicación entre nuestras máquinas definimos hostnames e IPs para cada una de nuestras máquinas de la siguiente manera:

|Máquina|Hostname|IP         |
--------|--------|-----------|
|**vm1**|server1 |192.168.0.3|
|**vm2**|server2 |192.168.0.4|
|**vm3**|server3 |192.168.0.5|

Editamos el archivo `/etc/hosts` y añadimos las siguientes líneas en cada una de las máquinas virtuales:
```
192.168.0.3 server1
192.168.0.4 server2
192.168.0.5 server3
```

Debemos averiguar el nombre de la interfaz que está conectada a nuestra red interna, esto lo podemos lograr con el siguiente comando:
```
ip address
```

En nuestro caso para la **vm1** la interfaz asignada es `enp0s8`.

Editamos el archivo de configuración de red `/etc/netplan/*.yaml` (en nuestro caso `/etc/netplan/50-cloud-init.yaml`) y configuramos la interfaz para cada una de nuestras máquinas de la siguiente manera:
<pre>
...
        <b>interfaz_intranet</b>:
            dhcp4: no
            addresses: [<b>ip_por_asignar</b>/24]
...
</pre>

En nuestro caso para la **vm1** quedaría:
<pre>
network:
    ethernets:
        enp0s3:
            dhcp4: true
        <b>enp0s8</b>:
            dhcp4: no
            addresses: [<b>192.168.0.3</b>/24]
    version: 2
</pre>


Aplicamos la configuración:
```
sudo netplan apply
```

## Instalación
A continuación, se describe la instalación del sistema en sí, con el cuál se podrán cargar, almacenar y visualizar los datos.

El diagrama de despliegue para este sistema es el siguiente:

![Diagrama de despliegue para el sistema](./diagrama_de_despliegue.png)

### Descarga y copia de los archivos necesarios

Para conseguir la ejecución del sistema se deben descargar los directorios *vm1*, *vm2* y *vm3* de este repositorio. Para ello se debe descargar el repositorio, para más información acerca de cómo descargar este repositorio visitar: [Cómo clonar un repositorio](https://docs.github.com/en/github/creating-cloning-and-archiving-repositories/cloning-a-repository).

Se sebe guardar en cada una de las máquinas el directorio correspondiente:

|Máquina|Directorio|
|-------|----------|
|**vm1**|vm1       |
|**vm2**|vm2       |
|**vm3**|vm3       |

### Instalación inicial

#### Ejecución de Elasticsearch y Kibana

Para configurar en un inicio nuestro sistema es importante que ejecutemos primeramente Docker Compose en las máquinas **vm1** y **vm2**, ingresando el siguiente comando en los directorios *vm1* y *vm2* respectivamente:
```
sudo docker-compose up -d
```

Esperamos unos minutos a que levante Elasticsearch y Kibana.

Podemos comprobar el estado de Elasticsearch con el siguiente comando en **vm1**:
```
curl -XGET localhost:9700
```

Cuando Elastic esté listo obtendremos una respuesta similar a la siguiente:
```
{
  "name" : "node1",
  "cluster_name" : "uaoelk",
  "cluster_uuid" : "DMBFiCgZSw-6qfupvh3egA",
  "version" : {
    "number" : "7.6.1",
    "build_flavor" : "default",
    "build_type" : "docker",
    "build_hash" : "aa751e09be0a5072e8570670309b1f12348f023b",
    "build_date" : "2020-02-29T00:15:25.529771Z",
    "build_snapshot" : false,
    "lucene_version" : "8.4.0",
    "minimum_wire_compatibility_version" : "6.8.0",
    "minimum_index_compatibility_version" : "6.0.0-beta1"
  },
  "tagline" : "You Know, for Search"
}
```

Una vez que levante Elasticsearch podemos comprobar el estado de Kibana ejecutando el siguiente comando en **vm2**:
```
curl -XGET localhost:5601
```

En este caso cuando Kibana esté listo no obtendremos una respuesta, si recibimos alguna, por ejemplo:
```
curl: (56) Recv failure: Connection reset by peer
```

Significa que todavía no está listo el servidor.


#### Ingresar a Kibana

Kibana se encuentra corriendo en la máquina **vm2** en el puerto 5601. Para poder acceder a ella desde un navegador podemos asignar la siguiente regla de reenvío de puertos en **VirtualBox**:

|Protocolo|IP anfitrión|Puerto anfitrión|IP invitado|Puerto invitado|
|---------|------------|----------------|-----------|---------------|
|TCP      |127.0.0.1   |4444            |           |5601           |

Esto nos permitirá acceder a Kibana a través de la dirección <http://localhost:4444> desde nuestro navegador.

#### Configuración del índice *volcanoes_data*

Para que nuestros datos sean cargados correctamente a Elasticsearch, especialmente si queremos visualizar en el mapa nuestra información, debemos crear el índice y configurarlo con un *geo-punto* antes de correr Logstash.

Para ello ingresamos a Kibana a través del navegador y en el menú del lado izquierdo seleccionamos **Dev Tools**. Escribimos o copiamos la siguiente consulta en la consola y la ejecutamos dándole al botón de play:

```
PUT volcanoes_data
{
  "mappings": {
    "properties": {
      "location": {
        "type": "geo_point"
      }
    }
  }
}
```

Podemos comprobar que el índice se haya creado al listar nuestros índices con la siguiente petición:
```
GET _cat/indices
```

**Nota:** En caso de haber corrido Docker Compose en la máquina **vm3** antes de esto es probable que se haya creado el índice sin el *geo-punto*, si es así, debemos recrearlo y volver a correr Logstash. Para eliminar el índice ejecutamos la siguiente consulta desde Kibana: `DELETE volcanoes_data` además de ejecutar el script para limpiar el puntero de filebeat en la máquina **vm3** (`./vm3/filebeat/filebeat_data/limpiar.sh`).

### Iniciar los contenedores

Debemos ingresar en cada una de las máquinas al directorio **vm\*** correspondiente y ejecutar Docker Compose:
```
sudo docker-compose up -d
```
Esto nos creará los siguientes contenedores:
|Máquina|Contenedores|
|-------|------------|
|**vm1**|es1 <br> es2 <br> es3 <br> metricbeat|
|**vm2**|kibana <br> metricbeat               |
|**vm3**|logstash-metricbeat <br> logstash-filebeat <br> metricbeat <br> filebeat|

Si queremos revisar los logs de algún contenedor en específico debemos ejecutar el siguiente comando en la máquina correspondiente:
<pre>
sudo docker logs -f <em>nombre_del_contenedor</em>
</pre>

Para dejar de seguir los logs debemos presionar <b>Ctrl + C</b>.

Esto puede tardar algunos minutos en levantar y posteriormente ingresar los datos a Elasticsearch.

Para mejorar el rendimiento de nuestra pila ELK podemos incrementar la memoria que le hemos asignado, para ello podemos editar los siguientes archivos:
```
vm1/elastic1_config/jvm.options
vm1/elastic3_config/jvm.options
vm1/elastic2_config/jvm.options

vm3/logstash_metricbeat_config/jvm.options
vm3/logstash_filebeat_config/jvm.options
```
Dentro de ellos editamos los parámetros `-Xms` y `-Xmx`. Algunos ejemplos:
```
-Xms256m
-Xmx256m
```
```
-Xms1g
-Xmx1g
```

Si cambiamos esta configuración debemos reiniciar nuestros contenedores al parar docker-compose con el comando `sudo docker-compose down` y volver a correr el comando `sudo docker-compose up -d`. Es importante que estos comandos sean ejecutados en los directorios **vm\*** o alguno de sus subdirectorios para funcionar correctamente.

### Parar los contenedores

Para detener la ejecución de nuestros contenedores debemos ejecutar el siguiente comando en el directorio **vm\*** de cada una de las máquinas:
```
sudo docker-compose down
```

### Cargar el Dashboard

El dashboard se encuentra en `kibana_dashboard/volcanoes_dashboard.ndjson`. Debemos cargar este archivo a Kibana. Para ello debemos seleccionar *Stack Management* en el menú de la izquierda, seleccionar *Saved Objects* y finalmente *Import*. Arrastramos el archivo `volcanoes_dashboard.ndjson` y damos click en *Import*. Listo, tenemos todos los objetos necesarios para ingresar y ver nuestros datos en el Dashboard.

### Visualizar el Dashboard

Para visualizar el Dashboard ingresamos en Kibana, seleccionamos **Dashboard** en el menú de la izquierda y seleccionamos *"Volcanoes Dashboard"*. Seleccionamos un periodo de tiempo apropiado (por ejemplo: los últimos 50 años) y recargamos. Si exsiten datos en el periodo de tiempo seleccionado nos mostrará nuestras visualizaciones.

![Dashboard de actividad volcánica](./dashboard_volcanoes.png)

Los íconos que se pueden observar fueron extraídos de [icons8.com](icons8.com).

¡Listo! Ahora sólo queda explorar los datos y, si lo deseas, puedes agregar más visualizaciones o modificar las existentes.